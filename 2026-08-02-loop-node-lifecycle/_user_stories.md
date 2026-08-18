# User Stories: Loop Node Lifecycle & Failure Contract

Canonical behavior catalog for the node lifecycle & failure contract (Spec 1 of the loop-hardening train). Companion to `_prd.md`; consumed by `_techspec.md` (component mapping) and `_tests.md` (coverage matrix).

## Personas

- **Loop author** — a developer writing loop definitions. Wants failure behavior expressible in a few lines, safe defaults when nothing is declared, and validation that catches mistakes before a run starts.
- **Authoring agent** — an agent that writes, validates, and repairs loop definitions programmatically. Depends on deterministic validation diagnostics that name what is wrong and what is available.
- **Operator** — a human watching runs through the web UI or command line. Needs to see what is failing, waiting, or quarantined, repair live runs without killing them, and trust that nothing rots invisibly.
- **Managing agent** — an agent operating loops through structured command/API surfaces. Needs every state observable and every verb invocable with structured output and deterministic errors, without any UI.
- **Approver** — a person who answers approval gates, usually from a notification link, and may not be the operator.
- **Extension developer** — builds extensions that provide tools and watch sources or consume lifecycle events. Needs truthful, ordered events and clear contracts for participating in reactions and admissions.

## Story Index

| ID     | Feature Area                  | Persona          | Story                                                              |
| ------ | ----------------------------- | ---------------- | ------------------------------------------------------------------ |
| US-001 | Classification & diagnostics  | Loop author      | Payload-declared failure detected as a real failure                |
| US-002 | Classification & diagnostics  | Authoring agent  | Authoring mistakes distinguished from runtime failures             |
| US-003 | Classification & diagnostics  | Loop author      | Remediation hints flow into repair and retry feedback              |
| US-004 | Classification & diagnostics  | Loop author      | Broken predicates fail by declared policy, always recorded         |
| US-005 | Failure handling & routing    | Loop author      | Route a node failure to a recovery path                            |
| US-006 | Failure handling & routing    | Loop author      | Explicitly absorb a non-critical failure                           |
| US-007 | Failure handling & routing    | Loop author      | Unannotated failure escalates into generation repair               |
| US-008 | Retry & time bounds           | Loop author      | Transient failures retry automatically on mechanical nodes        |
| US-009 | Retry & time bounds           | Loop author      | Expensive nodes never blind-retry; opt-in resumes from progress    |
| US-010 | Retry & time bounds           | Loop author      | Opt-in time limits with classified, routable timeout failures      |
| US-011 | Reactions                     | Loop author      | One-line failure notification with full context                    |
| US-012 | Reactions                     | Approver         | Approval request arrives where I work, with a usable link          |
| US-013 | Reactions                     | Loop author      | Loop-level reactions on every terminal outcome                     |
| US-014 | Reactions                     | Extension dev    | Lifecycle events are truthful, ordered, and deduplicable           |
| US-015 | Long-running supervision      | Operator         | Days-long nodes run without time-based death                       |
| US-016 | Long-running supervision      | Operator         | Session death auto-resumes from the last progress checkpoint       |
| US-017 | Long-running supervision      | Operator         | True prolonged silence flags attention without killing anything    |
| US-018 | Long-running supervision      | Managing agent   | Control verbs reach in-flight long work cooperatively              |
| US-019 | Pause & durable wait          | Operator         | Pause one node live, repair, resume — run survives                 |
| US-020 | Pause & durable wait          | Operator         | Rule-driven auto-pause turns retry storms into parked nodes        |
| US-021 | Pause & durable wait          | Loop author      | Durable waits survive restarts with governed resume                |
| US-022 | Pause & durable wait          | Approver         | Waiting approvals are visible with age; escalation when authored   |
| US-023 | Target health & quarantine    | Operator         | One sick dependency degrades one lane, not the run                 |
| US-024 | Target health & quarantine    | Operator         | Quarantined nodes carry repair context and requeue cleanly         |
| US-025 | Admission dedupe              | Loop author      | One delivered event starts exactly one run                         |
| US-026 | Manageability & configuration | Managing agent   | Every new state and verb is agent-operable with parity             |
| US-027 | Manageability & configuration | Operator         | Family defaults are configurable and validated                     |
| US-028 | Web authoring                 | Loop author      | Failure contract authored truthfully in the loop editor            |
| US-029 | Web hero path                 | Operator         | Catalog → run form → loop detail with truthful lifecycle states    |

## Classification & diagnostics

### US-001: Payload-declared failure detected as a real failure

**As a** loop author, **I want** a tool result whose body declares failure to be treated as a node failure with the real error message, **so that** routing and retry act on the truth instead of a transport-level "success".

Acceptance criteria:

- AC-1: Given a tool node whose call succeeds at the transport level, when the result body carries a declared error, then the node is failed with the body's error message as the failure cause.
- AC-2: Given a result body that declares unsuccessful completion without an error message, when the node completes, then the node is failed with a generic declared-failure cause naming the field that triggered it.
- AC-3: Given a node with a declared result contract naming a custom failure field, when the result matches that contract's failure shape, then the node fails per the contract; the built-in detection rules apply when no contract is declared.
- AC-4: Given a payload-declared failure, when retry policy is evaluated, then the failure classifies as semantic and is never retried automatically (ADR-003).

Edge cases:

- EC-1: Result body has both an error field and a success indicator → the error field wins (fixed precedence); the recorded cause names the winner.
- EC-2: Error field present but empty/null → not a failure; empty error containers (empty object/list) → failure with a "declared error was empty" cause.
- EC-3: Result body is not inspectable (binary/oversized) → detection skips with a recorded diagnostic; transport status decides.
- EC-4: Declared result contract itself is invalid (references fields the tool never produces) → authoring error at validation time, not a runtime surprise.
- EC-5: Failure message exceeds the diagnostic size bound → truncated with an explicit truncation marker; never dropped silently.

### US-002: Authoring mistakes distinguished from runtime failures

**As an** authoring agent, **I want** references to nonexistent fields to fail as authoring errors listing what is available, while references to skipped nodes resolve safely, **so that** I can repair definitions mechanically instead of guessing.

Acceptance criteria:

- AC-1: Given a reference to a field that does not exist on a node that ran, when the reference resolves, then an authoring-error diagnostic is raised naming the path, the node, and the available fields.
- AC-2: Given a reference to a node skipped by branching, when the reference resolves, then it yields an absent value without raising an error.
- AC-3: Given a definition with a resolvable-at-compile-time bad reference, when the definition is validated, then validation fails with the same diagnostic shape before any run starts.
- AC-4: Given diagnostics for authoring errors and runtime failures, when either is inspected on any surface, then its class (authoring vs runtime) is explicit and stable.

Edge cases:

- EC-1: Reference to a field on a quarantined or cancelled node → resolves against that node's last recorded output if one exists; absent value otherwise; never an authoring error.
- EC-2: Deeply nested path where only the leaf is wrong → diagnostic names the deepest valid segment and the available fields at that level.
- EC-3: Node renamed between generations of authoring → validation catches the stale reference; the diagnostic suggests existing node names.
- EC-4: Authoring error surfaced mid-run (dynamic reference) → the node fails as authoring-class, which never auto-retries and defaults to escalation.

### US-003: Remediation hints flow into repair and retry feedback

**As a** loop author, **I want** tool failures to carry actionable next-step hints that reach the repairing agent, **so that** repair generations act on guidance instead of raw error text.

Acceptance criteria:

- AC-1: Given a tool failure carrying a recovery hint, when the next generation runs with repair context, then the hint is present in that context alongside the failure cause.
- AC-2: Given a tool failure without a hint, when repair context is built, then a generic "review the failure and adjust" guidance appears instead of an empty slot.
- AC-3: Given a failure that leads to an automatic retry, when the retried node is an agent-facing one with authored retry, then the failure summary and hint are available to the retried attempt.

Edge cases:

- EC-1: Hint contains secrets or oversized content → redacted/bounded before it reaches any context or surface (same sanitization as all diagnostics).
- EC-2: Hint present but failure is absorbed by allow-fail → hint still recorded on the run for inspection, not injected anywhere.
- EC-3: Multiple failures in one generation with conflicting hints → each failure keeps its own hint; repair context lists them per node, never merged into one blob.

### US-004: Broken predicates fail by declared policy, always recorded

**As a** loop author, **I want** a broken continuation predicate to end the loop and a broken routing predicate to become a routable failure, **so that** a typo in an expression can neither hang a run forever nor pass silently.

Acceptance criteria:

- AC-1: Given a continuation predicate (stop/continue class) that throws or exceeds its evaluation bound, when it is evaluated, then the loop exits by policy and the exit reason names the broken predicate.
- AC-2: Given a routing predicate (branch/route class) that throws or exceeds its bound, when it is evaluated, then a routable failure is raised that error routes can catch, and unrouted it escalates normally.
- AC-3: Given any predicate evaluation failure, when the run is inspected, then a diagnostic records the predicate, the failure, and the policy outcome — including the continuation case (improvement over the Sim reference, which records nothing).
- AC-4: Given a predicate, when the definition is validated, then the expression compiles (compile-without-execute) or validation fails naming the expression.

Edge cases:

- EC-1: Predicate evaluation cost exceeds the bound at 80% → warning diagnostic; above 100% → treated as broken predicate under the kind's policy.
- EC-2: Author overrides the per-kind policy explicitly → override honored; both policies remain available per predicate.
- EC-3: Predicate references history namespaces on the first generation (no previous data) → resolves as absent data per namespace contract, not as a broken predicate.

## Failure handling & routing

### US-005: Route a node failure to a recovery path

**As a** loop author, **I want** to declare a forward error route on a fallible node, **so that** a failure runs my fallback (alternate tool, cleanup, degrade) without tripping run-level failure policy.

Acceptance criteria:

- AC-1: Given a node with an error route, when the node fails after its retries exhaust, then the error route activates, the success path stays inactive (mutually exclusive), and the failure is marked handled.
- AC-2: Given a handled failure, when run-level policy is evaluated, then it does not feed the target breaker and does not count as an unhandled failure — but it still counts toward iteration caps and budgets (ADR-002).
- AC-3: Given a definition whose error route points upstream (would create a cycle), when validated, then validation rejects it as a forward-only violation.
- AC-4: Given a failure with both an error route and failure reactions declared, when the node fails terminally, then reactions fire and the route is taken (reactions observe; routes direct flow).

Edge cases:

- EC-1: The error-route target itself fails → normal precedence applies to that node; if unhandled, it escalates to generation repair (no infinite route loops — routes are forward-only).
- EC-2: Error route declared on a node kind that cannot fail → validation warning naming the dead route.
- EC-3: Node succeeds → error route stays inactive and downstream-of-error references resolve as skipped (safe absent).
- EC-4: Failure occurs while a cancel is in flight → cancellation wins; the route does not activate; `on_cancel` reactions fire.
- EC-5: Two failures race in one generation across parallel lanes, each with routes → each routes independently; handled marks are per node.

### US-006: Explicitly absorb a non-critical failure

**As a** loop author, **I want** to declare that a node is allowed to fail, **so that** optional side work (best-effort enrichment) never blocks the run — while absorption stays impossible without my declaration.

Acceptance criteria:

- AC-1: Given a node declared allow-fail, when it fails, then the run continues, the failure is recorded as absorbed, and downstream references to its output resolve as absent.
- AC-2: Given a node without allow-fail and without an error route, when it fails, then the failure escalates (never silently absorbed) — absorption exists only by declaration.
- AC-3: Given an absorbed failure, when caps and budgets are evaluated, then the failure still counts toward them.

Edge cases:

- EC-1: Allow-fail node fails on every generation → absorbed each time; visible in run detail as a repeated absorbed failure; never trips the breaker (semantic/handled).
- EC-2: Allow-fail combined with an error route on the same node → validation rejects the ambiguity; author picks one.
- EC-3: Allow-fail on a node whose output later generations require via history namespaces → those references resolve absent; predicates over them follow the namespace contract.

### US-007: Unannotated failure escalates into generation repair

**As a** loop author, **I want** a failure I declared nothing about to flow into the generation repair loop with full context, **so that** doing nothing is safe and the engine's smartest recovery is the default.

Acceptance criteria:

- AC-1: Given a node with no failure annotations, when it fails (after default retries, if its family has them), then the generation ends and succession re-runs the producers with repair context carrying the classified failure and hints.
- AC-2: Given repeated unhandled failure of the same node across consecutive generations, when the failure-policy limit is reached, then existing run-level policy applies unchanged (breaker semantics per ADR-007 keying).
- AC-3: Given the precedence chain, when a failure occurs, then handling is attempted strictly in order — automatic retry, error route, reactions, escalation — and each step's outcome is observable in the run's record.

Edge cases:

- EC-1: Failure in the very first generation (no history) → repair context carries the failure with absent previous data; escalation works from generation one.
- EC-2: Failure of a node whose producers include a quarantined node → run surfaces needs-attention naming the quarantined dependency (US-024), never a silent stall.
- EC-3: Unhandled failure while the run has a pause request pending → generation completes its failure handling first; pause lands at the boundary as requested.

## Retry & time bounds

### US-008: Transient failures retry automatically on mechanical nodes

**As a** loop author, **I want** tool and transform nodes to retry transient failures by themselves with backoff, **so that** a network blip costs a delay instead of a generation.

Acceptance criteria:

- AC-1: Given a mechanical node with no retry configuration, when it fails with a transport/infrastructure-class cause, then it retries automatically with exponential backoff up to the family's small default cap.
- AC-2: Given a semantic failure (payload-declared, authoring error) or a cancellation, when retry is evaluated, then no automatic retry happens regardless of configuration defaults.
- AC-3: Given retries in progress, when the run is inspected on any surface, then the current attempt number, next-attempt time, and last failure are visible.
- AC-4: Given a failing attempt that states its own retry-after delay, when the next attempt is scheduled, then that delay is honored within the node's bounds.
- AC-5: Given retries exhaust, when the final failure lands, then it enters the normal precedence chain (route → reactions → escalation) exactly like a first failure would.

Edge cases:

- EC-1: Cancel arrives during backoff wait → pending retry is suppressed; `on_cancel` fires; no further attempts.
- EC-2: Pause (US-019) lands while a retry is pending → the retry does not fire while paused; resume options decide whether attempts reset.
- EC-3: Every attempt consumes caps/budgets — a retry storm cannot exceed run bounds (bounds are bounds).
- EC-4: Target breaker opens mid-retry-sequence → remaining attempts fail fast with the open-target cause; the sequence ends early.
- EC-5: Author sets attempts to zero → no automatic retry on that node; declared configuration always wins over family defaults.
- EC-6: Backoff computed past the node's opt-in total budget → the node closes with a budget-exhausted classification distinct from the last attempt's failure.

### US-009: Expensive nodes never blind-retry; opt-in resumes from progress

**As a** loop author, **I want** agent and sub-loop nodes to never re-run automatically on failure, and — when I opt in — to continue from their recorded progress, **so that** token spend stays intentional and crash recovery is continuation, not repetition.

Acceptance criteria:

- AC-1: Given an agent or sub-loop node with no retry configuration, when it fails, then no automatic retry occurs; the failure follows routes/reactions/escalation.
- AC-2: Given an agent node with authored retry, when a retried attempt starts and a progress checkpoint exists, then the attempt receives that checkpoint and continues from it instead of starting cold.
- AC-3: Given an opt-in retry on an expensive node, when attempts are accounted, then each attempt's token/work spend counts toward run budgets like any other work.

Edge cases:

- EC-1: Checkpoint exists but is unusable by the new attempt (declared incompatible by the work itself) → attempt starts cold and records that the checkpoint was declined.
- EC-2: Author opts an agent node into retry with a large cap → validation warns about unbounded expensive re-execution; caps/budgets still bound it at run level.
- EC-3: Failure is the session dying (infra) rather than the agent failing (semantic) → death-resume (US-016) applies, which is continuation and not a retry; the two paths are distinct in the record.

### US-010: Opt-in time limits with classified, routable timeout failures

**As a** loop author, **I want** to declare per-attempt and total time limits on nodes I know are short, **so that** a hung dependency turns into a classified, routable failure — while undeclared nodes stay unbounded by design.

Acceptance criteria:

- AC-1: Given a node with no declared limits, when it runs for any duration, then no time-based failure ever fires (ADR-004).
- AC-2: Given a declared per-attempt limit, when an attempt exceeds it, then the attempt fails with a timeout classification that is transient-class (retryable on mechanical nodes) and routable.
- AC-3: Given a declared total budget (attempts + waits between them), when it is exceeded, then the node closes with a budget-exhausted classification distinct from attempt timeout, entering the normal failure flow.
- AC-4: Given an invalid duration declaration, when the definition validates, then validation fails naming the field — never a silent fallback to some default.

Edge cases:

- EC-1: Timeout fires while the node is paused → it does not: paused/waiting states suspend declared node clocks (waiting is not work; ADR-005 posture applied at node level).
- EC-2: Attempt limit larger than total budget → validation rejects the contradiction.
- EC-3: Timeout and completion race → at-most-one outcome wins deterministically; a completed result arriving after timeout is recorded as late-arrival diagnostic, not a second outcome.
- EC-4: Stale timeout from a superseded attempt fires after the node moved on → ignored by construction (stale future work is invalidated when state changes — idea 27 as a product guarantee: no ghost timers).

## Reactions

### US-011: One-line failure notification with full context

**As a** loop author, **I want** to declare `on_error: notify my channel` on a node, **so that** terminal node failures reach me with the cause, the hint, and links — without wiring anything outside the loop.

Acceptance criteria:

- AC-1: Given a node with an `on_error` reaction, when the node fails terminally (after retries and routing resolve), then the reaction fires exactly once for that failure with the classified cause, remediation hint, node identity, and run link pre-bound.
- AC-2: Given `on_retry` declared, when each retry attempt is scheduled, then the reaction fires per attempt with the attempt number and next-attempt time.
- AC-3: Given a reaction whose effect fails (dead webhook, missing tool), when it runs, then the failure is recorded on the run and the node's outcome is untouched (fail-open).
- AC-4: Given multiple effects on one trigger, when they run, then each is isolated — one failing effect never stops the others — and their results are recorded per effect.

Edge cases:

- EC-1: Node is killed (not cancelled) → no reactions fire (kill is immediate by contract); the kill itself is a recorded lifecycle event.
- EC-2: Effect tool references a tool the workspace does not have → recorded effect failure with a deterministic missing-tool cause; never an authoring block at run time — but validation warns when the tool is unknown at authoring time.
- EC-3: Reaction fires during a crash window → after recovery, the effect is delivered at-least-once with stable identity so consumers can dedupe (US-014); it is never delivered for a state that did not durably land.
- EC-4: `on_error` on a node whose failure is absorbed by allow-fail → `on_error` still fires (the node did fail); the absorbed disposition is part of the payload.

### US-012: Approval request arrives where I work, with a usable link

**As an** approver, **I want** pause/approval reactions to deliver a link that opens the exact decision, **so that** I can approve from where I was notified without hunting through the UI.

Acceptance criteria:

- AC-1: Given a gate or wait with an `on_pause` reaction, when the run parks, then the notification carries a resume/approval link bound to that exact pause point (node, branch, iteration).
- AC-2: Given the link, when I open it, then I land on the decision surface for that pause with enough context to decide (what is being asked, by which loop/run, since when).
- AC-3: Given my decision, when it is submitted, then the run resumes per the decision and my identity and timestamp are recorded on the decision.

Edge cases:

- EC-1: Link used after the pause was already resolved → deterministic "already decided" answer showing who decided and when; never a second application.
- EC-2: Two approvers act concurrently → one decision wins (single-claim); the loser receives an explicit already-decided answer (US-021 governs resume admission).
- EC-3: Link shared with someone lacking permission → denied with a deterministic authorization error; the denial is auditable.
- EC-4: Notification effect failed (channel down) → the pause is still fully visible and decidable on operator surfaces; the effect failure is recorded (fail-open never hides the pause).

### US-013: Loop-level reactions on every terminal outcome

**As a** loop author, **I want** to declare reactions on the loop's terminal outcomes, **so that** "tell me how it ended, however it ended" is one line in the contract.

Acceptance criteria:

- AC-1: Given loop-level reactions declared for terminal outcomes, when the run reaches any terminal outcome (done, no-op, blocked, failed, exhausted, stalled, canceled), then the matching reaction fires exactly once per run with the outcome, the reason, and the run link.
- AC-2: Given no reaction for a specific outcome, when that outcome lands, then nothing fires and nothing errors (declaring a subset is valid).
- AC-3: Given a run that ends by operator cancel or kill, when terminal state lands, then the terminal reaction for the resulting outcome still fires (loop-level reactions ride terminal truth, not the path to it; node-level `on_*` reactions stay suppressed on kill).

Edge cases:

- EC-1: Run crashes and recovers exactly at terminal transition → the terminal reaction is delivered at-least-once after recovery, never zero times, never for a non-terminal state.
- EC-2: Reactions declared on both a node and the loop for the same underlying failure → both fire in their own scopes (node-level once for the node, loop-level once for the run); payloads distinguish scope.
- EC-3: Terminal outcome no-op (watch loop with nothing to do) → reaction fires with no-op semantics; authors can use it for "heartbeat: still alive, nothing to do" signals.

### US-014: Lifecycle events are truthful, ordered, and deduplicable

**As an** extension developer, **I want** node/loop lifecycle events (including reaction emissions) to arrive only after the state they announce is durably recorded, with stable identity, **so that** my consumers never act on phantom states and can dedupe replays.

Acceptance criteria:

- AC-1: Given any lifecycle transition (failure, retry scheduled, pause, resume, cancel, quarantine, terminal), when the corresponding event is observable anywhere (hooks, live stream, reaction emissions), then the underlying state is already durable — an event is never observable before its state.
- AC-2: Given crash/recovery around a transition, when events are delivered, then delivery is at-least-once with a stable event identity sufficient for consumer-side dedupe.
- AC-3: Given a subscriber that fails while handling an event, when the failure occurs, then the run is unaffected (observer isolation — the Mastra watch anti-lesson inverted).

Edge cases:

- EC-1: Consumer replays history from the durable record → replayed events match what was originally emitted (no divergence between live and replayed truth).
- EC-2: Burst of transitions (retry storm on many nodes) → events remain ordered per node; cross-node ordering follows recorded history order.
- EC-3: Reaction-emitted custom events → same contract: post-commit, stable identity, observer isolation.

## Long-running supervision

### US-015: Days-long nodes run without time-based death

**As an** operator, **I want** a goal-carrying agent node to run for days while I watch its progress, **so that** the platform's core workload never fights a hidden clock.

Acceptance criteria:

- AC-1: Given an agent node with no declared limits, when it runs for hours or days while showing signs of life, then nothing interrupts it and no warning fires based on duration alone.
- AC-2: Given a long-running node, when I inspect the run, then I see its latest progress checkpoint, the checkpoint's age, and the liveness status (alive via stream/tool activity).
- AC-3: Given a quiet-but-working phase (a long tool execution producing no output), when liveness is evaluated, then the in-flight tool execution counts as activity and no silence flag is raised.

Edge cases:

- EC-1: Run-level budgets (tokens, authored wall-clock) still bound the run — working time counts, waiting time does not (ADR-005); budget exhaustion follows the budget's declared policy, not a node timeout.
- EC-2: Operator watches during a transport reconnect blip → brief signal loss is not death (death is confirmed, not inferred); the UI shows degraded signal, not a dead node.
- EC-3: Progress checkpoints stop advancing while streams stay active → node remains healthy by liveness; "active but not progressing" is visible in progress metadata for human judgment (and is the held idea 12's future evidence base).

### US-016: Session death auto-resumes from the last progress checkpoint

**As an** operator, **I want** a confirmed-dead agent session to resume automatically from its last checkpoint, **so that** a 3 a.m. crash costs minutes of overlap, not a restart of days of work.

Acceptance criteria:

- AC-1: Given a long node whose session death is confirmed (process/transport definitively gone), when the engine reacts, then a resume attempt starts from the last recorded checkpoint automatically, marked as continuation (distinct from retry) with visible provenance.
- AC-2: Given repeated confirmed deaths, when the bounded resume cap is exhausted, then the node surfaces as needs-attention with the death/resume history — never an infinite resume loop.
- AC-3: Given a resume, when it continues, then prior progress is not re-executed (continuation semantics) and accounting continues cumulatively.

Edge cases:

- EC-1: Death while the node is paused → no resume (paused means parked); the death is recorded and visible; resume-from-death applies only to live work when unpaused.
- EC-2: Death while awaiting approval → the wait is unaffected (waits don't need the session); the session resumes when work actually continues.
- EC-3: No checkpoint exists yet (death right after start) → resume starts from the beginning; still bounded by the resume cap.
- EC-4: Death and operator cancel race → cancel wins; the node closes cancelled, no resume fires.
- EC-5: A false "death" (transport blip misread) must be impossible by contract: only confirmed, deterministic death triggers resume — ambiguity resolves to the silence path (US-017), never to resume.

### US-017: True prolonged silence flags attention without killing anything

**As an** operator, **I want** nodes with no evidence of life for a long, configurable window to be flagged for attention, **so that** wedged sessions surface while slow-but-alive work is never punished.

Acceptance criteria:

- AC-1: Given a node with no stream activity, no in-flight tool execution, and no transport presence for the configured window, when the window elapses, then the node is flagged needs-attention with "no evidence of life since …" — and nothing else happens automatically.
- AC-2: Given the flag, when any sign of life returns, then the flag clears automatically and the episode is recorded.
- AC-3: Given a flagged node, when I inspect it, then I see the last evidence of life, the window, and the available verbs (cancel, kill, pause) — the decision is mine.

Edge cases:

- EC-1: `make gate-full`-class tool execution running many minutes with no output → never flagged: an in-flight tool execution is evidence of life by definition (hard requirement from grilling).
- EC-2: Window configured to zero/disabled → no silence evaluation at all; death-resume (US-016) still works (they are independent).
- EC-3: Flag raised, then session death confirmed → death path takes over (auto-resume); the silence episode closes into the timeline.
- EC-4: Many nodes flagged at once (workspace-wide outage) → flags aggregate visibly per run and per workspace; no flag storm hides the root cause.

### US-018: Control verbs reach in-flight long work cooperatively

**As a** managing agent, **I want** cancel and pause to reach a running node through its own liveness channel, **so that** live work reacts to control without a second delivery path — and silent work is still stoppable by kill.

Acceptance criteria:

- AC-1: Given a live node reporting liveness, when cancel or pause is issued, then the in-flight work observes the request on its next liveness exchange and begins cooperative handling (drain, checkpoint, close) within the grace expectations.
- AC-2: Given cooperative cancel completes, when the node closes, then `on_cancel` reactions ran, cleanup ran, pending retries were suppressed, and the close reason distinguishes cancel from failure.
- AC-3: Given work that never reports liveness, when cancel cannot be delivered, then the verb's response says so and kill remains available and immediate.

Edge cases:

- EC-1: Cancel issued twice / cancel after terminal → idempotent, deterministic answers (no error storm; second cancel reports current state).
- EC-2: Cancel racing completion → completion may win; the response reports the actual outcome (completed, not cancelled).
- EC-3: Grace exceeded without cooperative close → nothing silent happens: the state is visible and kill is the operator/agent's explicit escalation (no hidden auto-kill default, per ADR-008).

## Pause & durable wait

### US-019: Pause one node live, repair, resume — run survives

**As an** operator, **I want** to pause a wedged-on-bad-credential node while the rest of the run continues, fix the credential, and resume, **so that** one bad dependency doesn't cost me a days-long run.

Acceptance criteria:

- AC-1: Given a live run, when I pause a node, then that node stops being scheduled (in-flight attempt finishes or is cancelled per my choice), the rest of the run proceeds, and the pause carries provenance (who, why, when, manual-vs-rule).
- AC-2: Given a paused node, when stall/no-progress policy is evaluated, then the paused node is excluded (waiting, not failing) while staying visible in progress reporting.
- AC-3: Given resume, when I choose plain resume vs resume-with-attempt-reset vs resume-immediate, then the chosen semantics apply and are recorded.
- AC-4: Given pause/resume/reset verbs, when invoked from any surface (UI, command line, API, agent tools), then behavior and structured responses are identical.

Edge cases:

- EC-1: Pause a node that is mid-retry-backoff → pending retry parks; resume decides whether the attempt counter resets.
- EC-2: Pause an already-paused node / resume a non-paused node → idempotent success / deterministic invalid-state answer.
- EC-3: Pause requested on a terminal run → deterministic invalid-state answer naming the terminal outcome.
- EC-4: Everything pauses (all lanes paused) → run state shows paused-dominant status; nothing counts toward stall; budgets' work clock suspends.
- EC-5: A failure occurs while its node is paused (late async result) → recorded against the paused node without waking it; handling runs at resume.

### US-020: Rule-driven auto-pause turns retry storms into parked nodes

**As an** operator, **I want** configured rules that pause matching nodes automatically (by failure class, attempt count, target), **so that** a crash-looping dependency parks recoverably instead of burning budget.

Acceptance criteria:

- AC-1: Given an auto-pause rule matching a node's failure pattern, when the pattern occurs, then the node pauses with rule provenance (rule identity, matched condition) instead of continuing to retry.
- AC-2: Given a rule-paused node, when I resume it, then normal resume semantics apply; the rule does not re-fire on the same episode unless the pattern recurs.
- AC-3: Given rules configuration, when a rule is written or updated, then it is validated (expression compiles, fields exist) and rejected with diagnostics when invalid.

Edge cases:

- EC-1: Rule matches during the first attempt (overly broad) → still honored, but visible provenance makes the broad rule diagnosable; rules can be narrowed live.
- EC-2: Two rules match one event → deterministic evaluation order; first match wins and is named in provenance.
- EC-3: Rule pauses a node whose output a gate re-attempt needs → same as any parked required producer: run surfaces needs-attention naming the rule pause.

### US-021: Durable waits survive restarts with governed resume

**As a** loop author, **I want** waits (timers, external events, approvals) to persist across engine restarts and resume exactly once, **so that** parked runs are as durable as running ones.

Acceptance criteria:

- AC-1: Given a parked wait, when the engine restarts, then the wait is still parked with its identity, age, and expected resumption; nothing is lost or duplicated.
- AC-2: Given a due timer wake or an arriving event/approval, when resume is admitted, then exactly one resume wins (single claim); concurrent attempts get deterministic queued/already-resolved answers.
- AC-3: Given a resume payload that violates the wait's declared expectation, when admission validates it, then it is rejected with a diagnostic naming the mismatch and the wait stays parked.
- AC-4: Given resume admission that keeps failing, when bounded admission attempts exhaust, then the wait surfaces as intervention-required — visible, never silent limbo.

Edge cases:

- EC-1: Wait inside a fan-out lane → wait identity is scoped to node + branch + iteration; two lanes' waits never collide.
- EC-2: Timer due while the run is paused → wake honors pause (parks the wake); fires on resume.
- EC-3: Event arrives before the run reaches the wait → per declared wait semantics (ahead-arrival recorded and consumed at wait entry, or rejected) — the choice is authorable, the default is consume-on-entry.
- EC-4: Restart during resume admission → claim semantics guarantee no double-resume; admission continues or re-runs idempotently.

### US-022: Waiting approvals are visible with age; escalation when authored

**As an** approver (and the operators around me), **I want** everything waiting on a human to be listed with its age, and authored escalation ladders to fire on my behalf, **so that** nothing rots invisibly and nothing expires behind my back.

Acceptance criteria:

- AC-1: Given any waiting approval/wait, when operator surfaces list waiting work, then it appears with loop, run, node, reason, and age — sortable and filterable; by default it waits indefinitely (no expiry).
- AC-2: Given an authored escalation ladder (re-notify → notify channel → route timeout path), when its declared deadlines pass, then each step fires in order via reactions, recorded on the run.
- AC-3: Given an authored terminal expiry with no declared timeout route, when it expires, then the wait surfaces needs-attention rather than silently discarding work.

Edge cases:

- EC-1: Escalation reaction fails (channel down) → fail-open recording; the wait and its age remain visible regardless.
- EC-2: Decision arrives mid-escalation → decision wins; remaining ladder steps cancel; the record shows both.
- EC-3: Zero waiting items → the waiting inventory shows a truthful empty state (no phantom entries from resolved waits).

## Target health & quarantine

### US-023: One sick dependency degrades one lane, not the run

**As an** operator, **I want** repeated transport failures against one target to open that target's breaker only, **so that** nodes on healthy targets keep working while the sick one fails fast with a clear cause.

Acceptance criteria:

- AC-1: Given repeated transport-class failures against one target, when its breaker opens, then subsequent nodes bound to that target fail fast with an explicit target-unavailable cause that follows the normal failure chain (routes/reactions/escalation).
- AC-2: Given other targets, when the breaker for one target is open, then nodes bound to healthy targets are unaffected — no loop-global stop from one target's sickness.
- AC-3: Given a recovery probe succeeding, when the target heals, then the breaker closes and normal scheduling resumes; open/close transitions are recorded and observable.
- AC-4: Given semantic failures ("the tool said no") or handled failures, when breaker accounting runs, then they never count toward opening.

Edge cases:

- EC-1: All targets of a run open at once → the run degrades to whatever can proceed; if nothing can, normal escalation/stall policy takes over with target-unavailable causes (diagnosable, not mysterious).
- EC-2: Breaker state across runs → target health is shared where the target is shared; a run inspecting its failure sees which target and since when.
- EC-3: Open breaker vs pending retries → remaining retries against an open target fail fast (US-008 EC-4) instead of sleeping through backoff.

### US-024: Quarantined nodes carry repair context and requeue cleanly

**As an** operator, **I want** a repeatedly-failing node to quarantine with everything I need to fix it, and a requeue verb to re-enter it after repair, **so that** repair-and-continue replaces run death.

Acceptance criteria:

- AC-1: Given a node that exhausts unexpected-failure handling, when it quarantines, then the run continues on independent branches and the quarantine entry carries the classified failure chain, attempt history with timestamps, remediation hints, involved target, and the node's input reference.
- AC-2: Given quarantine inventory, when I list it (per run, per loop, per workspace), then entries are inspectable and requeueable with structured output on every surface.
- AC-3: Given requeue after repair, when the node re-enters, then it does so through normal generation succession, all caps and budgets still applying; requeue provenance is recorded.
- AC-4: Given forward progress requiring a quarantined node (including a gate-driven re-attempt), when the requirement is hit, then the run surfaces needs-attention naming the quarantined dependency — never silent waiting, never cryptic failure.

Edge cases:

- EC-1: Requeue on a run that meanwhile reached terminal → deterministic invalid-state answer; entry remains inspectable for post-mortem.
- EC-2: Requeue while the target's breaker is still open → admitted but fails fast with the open-target cause (visible immediately), or the operator sees breaker state on the entry before requeueing.
- EC-3: Repeated quarantine of the same node after requeue → each episode appends to the entry's history; no cap bypass across episodes.
- EC-4: Quarantine of a node whose reactions declared `on_quarantine` → the reaction fires with the entry context (notification with repair link).
- EC-5: Quarantine entry containing secrets in inputs/failures → sanitized like all diagnostics; the inspecting surface never leaks raw credentials.

## Admission dedupe

### US-025: One delivered event starts exactly one run

**As a** loop author, **I want** duplicate deliveries of the same event to be suppressed loudly at admission, **so that** redelivery after restarts or source retries never double-runs my loop.

Acceptance criteria:

- AC-1: Given a watch-triggered loop, when the same event (same source, same event identity) is delivered more than once, then exactly one run starts; every duplicate is suppressed with a recorded diagnostic and a structured duplicate answer to the deliverer.
- AC-2: Given a restart replaying deliveries, when replay occurs, then previously admitted events stay suppressed (durable tombstones) within the source's redelivery horizon.
- AC-3: Given a watch source that cannot yield a stable event identity, when the definition validates, then validation fails naming the missing identity — the loop never runs undeduplicated (no silent fallback, no random keys).

Edge cases:

- EC-1: Two distinct events arrive while a run is live → both are distinct admissions; what happens next (overlap policy) is explicitly out of scope for Spec 1 and recorded as such — no accidental semantics.
- EC-2: Duplicate arrives after the original run reached terminal → still suppressed within the horizon; the diagnostic references the original run.
- EC-3: Extension-provided watch source declares identity fields → same contract applies to extensions; a source shipping without stable identity fails the same validation.
- EC-4: Suppression storm (source stuck re-sending) → suppressions are visible and countable per source, making the sick source diagnosable instead of silently absorbing load.

## Manageability & configuration

### US-026: Every new state and verb is agent-operable with parity

**As a** managing agent, **I want** every state and verb this contract introduces to be fully operable through structured surfaces, **so that** I can supervise and repair loops with no UI involved.

Acceptance criteria:

- AC-1: Given the new states (retrying with attempt info, paused with provenance, waiting with age, needs-attention with reason, quarantined with context) and verbs (node pause/resume/reset, cancel, kill, requeue, waiting/quarantine listing), when accessed via command line, API, and agent tools, then all exist with parity: same semantics, structured output, and stable field names.
- AC-2: Given an invalid invocation (wrong state, unknown node, terminal run), when it is answered, then the error is deterministic and machine-readable, naming the actual state and the allowed transitions.
- AC-3: Given list surfaces (waiting inventory, quarantine inventory, attention flags), when queried with filters, then results are consistent with what the UI shows (one truth).

Edge cases:

- EC-1: Verb issued concurrently by an agent and a human → single-claim semantics; the loser receives the deterministic already-applied answer with the winner's provenance.
- EC-2: Agent operates a run in another workspace → existing cross-workspace policy applies unchanged (this contract adds no new crossing).
- EC-3: Structured outputs under scale (hundreds of waiting/quarantined entries) → paginated, stable ordering, no truncation without indication.

### US-027: Family defaults are configurable and validated

**As an** operator, **I want** the contract's family defaults (retry caps and backoff, liveness windows, breaker thresholds, auto-pause rules) to be configurable at the loop-defaults level with validation, **so that** I can tune the platform to my workloads without editing every loop.

Acceptance criteria:

- AC-1: Given configuration surfaces for loop defaults, when I set retry/liveness/breaker/auto-pause defaults, then they validate on write (rejecting contradictions with named diagnostics) and apply to subsequent runs.
- AC-2: Given per-loop or per-node declarations, when they conflict with configured defaults, then the more specific declaration wins deterministically (node over loop over defaults).
- AC-3: Given configuration inspection, when I or an agent read effective settings for a loop, then the resolved values and their source (default vs override) are visible.

Edge cases:

- EC-1: Configuration lowered mid-flight (e.g., retry cap reduced) → running runs keep their admission-time resolution; new runs get the new defaults (no mid-run rule swaps).
- EC-2: Invalid auto-pause rule expression → rejected at write with the compile diagnostic; existing valid rules stay untouched.
- EC-3: Defaults removed entirely → engine's shipped defaults apply; the resolution chain never resolves to "nothing".

## Web authoring

### US-028: Failure contract authored truthfully in the loop editor

**As a** loop author, **I want** to declare the Spec 1 failure contract (retry, error absorption, reactions, waits, parent-close) in the loop editor with the same validation the daemon enforces, **so that** I can publish a loop whose failure behavior matches what the runtime will run — without leaving the UI or inventing fields the engine does not model.

Acceptance criteria:

- AC-1: Given a custom loop in the editor, when I edit the reliability envelope (`deadline`, `retry` with backoff/non_retryable, `result_contract`, `on_error` with route XOR allow_fail plus effects), then the values round-trip through save/publish and reappear on reopen exactly as stored.
- AC-2: Given node triggers and contract terminals, when I author effect lists (`emit`/`tool` + `with`), then the UI enforces effect one-of emit/tool and shows the seven terminals including `canceled`; empty effect lists omit zero-count chrome.
- AC-3: Given a `wait` node and a run-loop node, when I open their inspectors, then I can set wait modes (`for`/`until`/`event`), `expect`, `ahead_arrival`, `expires`, and `on_parent_close` (`terminate`/`cancel`/`abandon`).
- AC-4: Given validation diagnostics, when the dock has errors, then Publish is disabled; when it has only warnings (including `wait_expiry_without_path`), then Publish stays enabled and the warning remains visible; a zero issue count renders no counter.
- AC-5: Given a built-in loop, when I open the editor, then the chrome is read-only with a fork-before-publish path and Publish disabled; Validate remains available.
- AC-6: Given a publish that the daemon rejects (422), when the response returns, then a publish-rejected strip shows the issue list and the version does not advance.

Edge cases:

- EC-1: Invalid input — `on_error` with both `route` and `allow_fail` set → UI and/or validation blocks Publish with a named diagnostic; neither silent accept nor invented third mode.
- EC-2: Empty / missing — node with no `on_*` triggers and no reliability override → editor shows calm empty folds (no zero badges); publish succeeds; runtime keeps Spec 1 defaults.
- EC-3: Permissions — operator without edit rights / cross-workspace loop → existing workspace policy applies; editor does not invent a bypass.
- EC-4: Concurrency — two authors publish the same expected version → loser sees publish-rejected / version conflict truth; winner's version advances once.
- EC-5: State transitions — opening a terminal/archived definition → deterministic read-only or not-found; no editable chrome on a non-editable revision.
- EC-6: Start-binding allowlist chrome drawn on the Visual Contract artboard → out of scope for Spec 1 (ADR-018 / Spec 3); production keeps current Start strip behavior; this story does not require Start lane write path.

## Web hero path

### US-029: Catalog → run form → loop detail with truthful lifecycle states

**As an** operator, **I want** to find a loop, start a run, and read its contract and recent runs through the redesigned catalog → run form → detail path, **so that** arrive-and-use matches the Visual Contract and every status I see (including `canceled`) is daemon truth.

Acceptance criteria:

- AC-1: Given the loops catalog, when it loads, then built-in and custom groups render with search, status filter including `canceled`, Rows|Cards layout, truthful empty/clear-filters states, and a primary Start run action.
- AC-2: Given Start run from the catalog (or equivalent deep link), when the run form opens, then inputs are open, limits are folded but informative, Dry run and Start run are present, Start is gated on required inputs, and no `stop` control exists.
- AC-3: Given the run form's "Ways to start" truth, when I read it, then this form is identified as `http` and other declared start kinds appear as text only (authoring those kinds is not this story).
- AC-4: Given the loop detail page, when it loads, then Contract, reliability tiles (plain-language Spec 1 posture), steps DAG, and recent runs (including a `canceled` pill when present) render from daemon payloads.
- AC-5: Given the same loop entity across catalog, run form, and detail, when I navigate the hero path, then identity and status chrome stay consistent (one-story continuity).

Edge cases:

- EC-1: Empty / missing — catalog filters match nothing → truthful empty state with clear-filters affordance; no fabricated rows.
- EC-2: Invalid input — required run input blank or malformed → Start stays disabled or submit rejects with a field-named message; no run created.
- EC-3: Ordering / deep link — open run form or detail for an unknown loop id → deterministic not-found; no partial hero chrome.
- EC-4: State transitions — loop with only `canceled` recent runs → status filter and pills show `canceled`; no `stop` or invented terminal.
- EC-5: Scale — many loops / many recent runs → list remains usable (existing pagination/virtualization patterns); no silent truncation without indication.
- EC-6: Active run notice on the run form when a run is already live → informative open-run notice; does not invent pause/stop controls beyond what the runtime exposes on that surface.
