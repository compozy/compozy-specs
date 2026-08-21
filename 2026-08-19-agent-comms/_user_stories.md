# User Stories: Agent Comms — Typed Calls, Mailbox, and Subagents

Canonical behavior catalog for agent-comms. Companion to `_spec.md`; consumed by
`_spec.md` Part II (component mapping), `_uiux.md` (surface states), and
`_tests.md` (coverage matrix).

Vocabulary (glossary candidates, finalized in the surface contract): a **call** is a typed
delegation from a caller to a target agent — it carries the first prompt and an optional
**result contract** (the declared shape of a valid result). A **subagent** is a child session
running a named agent definition on the caller's behalf. The **mailbox** is the durable
message path that exists automatically between a parent and its children. A **parked**
child has finished working but remains addressable until its TTL. This feature is never
called a "handoff" — the caller always retains control.

## Personas

- **Agent Author** — a developer who writes agent definitions and Loops; declares result contracts and expects delegation to be reliable without prompt engineering.
- **Parent Agent** — an LLM session (coordinator, Loop-bound worker, or interactive session) that delegates work and consumes typed results through native tools.
- **Child Agent / Subagent** — an LLM session executing delegated work; must return results, ask questions, and report blockers through structured surfaces.
- **Operator** — the human running CompozyOS; watches delegation trees in the OS shell, intervenes (message, cancel, revive), and audits cost and provenance.
- **Automation Integrator** — drives calls headlessly through CLI/HTTP with structured output and deterministic errors (CI, scripts, external orchestrators).
- **Extension Author** — extends the runtime; needs call/message lifecycle hooks and stable contracts to build on.

## Story Index

| ID     | Feature Area   | Persona                | Story                                                        |
| ------ | -------------- | ---------------------- | ------------------------------------------------------------ |
| US-001 | Typed Call     | Parent Agent           | Delegate work with a result contract in one verb             |
| US-002 | Typed Call     | Parent Agent           | Follow up with an existing child using the same verb         |
| US-003 | Typed Call     | Parent Agent           | Fan out a batch of tasks in one call                         |
| US-004 | Typed Call     | Automation Integrator  | Retry a call safely with caller-chosen idempotency           |
| US-005 | Typed Call     | Parent Agent           | Cancel an in-flight call and see it actually stop            |
| US-006 | Typed Call     | Parent Agent           | Await a call result explicitly, bounded and resumable        |
| US-007 | Typed Return   | Child Agent            | Return a typed result as the terminal act                    |
| US-008 | Typed Return   | Child Agent            | Get exactly one repair round on an invalid result            |
| US-009 | Typed Return   | Parent Agent           | Distinguish "completed without payload" from success/failure |
| US-010 | Typed Return   | Agent Author           | Declare one result budget with an explicit overflow strategy |
| US-011 | Typed Return   | Parent Agent           | Recover a result when the child answered only in prose       |
| US-012 | Mailbox        | Child Agent            | Message the parent mid-flight                                |
| US-013 | Mailbox        | Parent Agent           | Steer a running child without interrupting it                |
| US-014 | Mailbox        | Parent Agent           | See delivery receipts and outcomes for sent messages         |
| US-015 | Mailbox        | Operator               | Runaway message loops stop on their own                      |
| US-016 | Mailbox        | Operator               | Peer messages carry provenance and cannot escalate           |
| US-017 | Lifecycle      | Parent Agent           | Completion wake carries the typed result                     |
| US-018 | Lifecycle      | Parent Agent           | Revive a parked child by calling or messaging it             |
| US-019 | Lifecycle      | Parent Agent           | Calls past the child's TTL fail with a typed error           |
| US-020 | Lifecycle      | Operator               | Parent termination resolves outstanding calls predictably    |
| US-021 | Subagents      | Agent Author           | Describe an agent so callers can discover it                 |
| US-022 | Subagents      | Parent Agent           | Pick a subagent from the roster without a lookup call        |
| US-023 | Subagents      | Parent Agent           | Delegate recursively to depth 3, honestly walled             |
| US-024 | Subagents      | Automation Integrator  | Manage the agent roster through structured surfaces          |
| US-025 | Unification    | Agent Author           | Loop node outputs honor the declared contract at settle      |
| US-026 | Unification    | Agent Author           | Tasks declare a result contract enforced at completion       |
| US-027 | Unification    | Agent Author           | One validation and budget regime across every return path    |
| US-028 | Observability  | Operator               | See the delegation tree, inboxes, and results in one app     |
| US-029 | Observability  | Operator               | Session detail shows calls and why the session woke          |
| US-030 | Observability  | Operator               | Blocked and failed delegations surface as needs-you signals  |
| US-031 | Observability  | Operator               | Delegation cost is visible and truthful                      |
| US-032 | Governance     | Parent Agent           | Call-spawned children get subset-only permissions            |
| US-033 | Governance     | Operator               | Cross-workspace delegation stays hard-denied                 |
| US-034 | Governance     | Operator               | Secrets never ride result or message payloads                |
| US-035 | Network Bridge | Parent Agent           | Publish a completed call into a Network conversation         |
| US-036 | Extensibility  | Extension Author       | Observe the call and message lifecycle through hooks         |
| US-037 | Extensibility  | Extension Author       | Read calls and messages through the consented Host API       |
| US-038 | Governance     | Operator               | Operator calls converge on one stable workspace identity     |

## Typed Call

### US-001: Delegate work with a result contract in one verb

**As a** Parent Agent, **I want** one verb that names a target agent, carries my prompt, and declares the shape of the result I expect, **so that** delegation is a single act and the answer I get back is machine-usable without parsing prose.

Acceptance criteria:

- AC-1: Given a target agent definition name, a prompt, and an optional result contract, when I invoke the call verb, then a child session is created under my lineage governance (TTL, permission narrowing, caps) and the call is durably recorded before I receive its identity.
- AC-2: Given the call was accepted, when I inspect the verb's structured response, then it contains the call identity, the child's identity, and the call state — and returns without waiting for the child to finish.
- AC-3: Given I declared a result contract, when the child completes, then the recorded result is guaranteed valid against that contract or the call ends in a typed failure state — a completed call never carries a silently invalid result.
- AC-4: Given I omit the result contract, when the child completes, then the call completes with the child's prose answer as the result, marked as uncontracted.
- AC-5: Given any call, when I list my calls through the native tool, CLI, or API, then I see the same states and identities on every surface.

Edge cases:

- EC-1: Unknown agent definition name → typed error naming the unknown value and the available roster; no session is created. (Invalid input)
- EC-2: Malformed result contract (not a valid schema) → typed error at call time, before any spawn; the contract is never accepted "for later". (Invalid input)
- EC-3: Empty prompt → typed error; a call always carries work. (Empty / missing)
- EC-4: Caller at its live-children cap → typed error naming the cap and current usage; nothing spawns. (Limits)
- EC-5: Caller's permission set cannot cover the target definition's requirements → typed error naming the widening atoms. (Permissions)
- EC-6: Daemon restarts after the call record commits but before the child starts → the call survives and is recovered or terminally failed with a typed reason; it never vanishes. (Interruption)
- EC-7: Two identical calls without idempotency keys → two independent calls, two children; the surfaces show both. (Repetition)

### US-002: Follow up with an existing child using the same verb

**As a** Parent Agent, **I want** the same call verb to address a child that already exists (running or parked), **so that** first contact and follow-up are one mental model and the child's context is preserved.

Acceptance criteria:

- AC-1: Given a live child of mine, when I call it by its identity with a new prompt, then no new session is created and the work is delivered to that child as a new call with its own identity and (optional) result contract.
- AC-2: Given a parked child, when I call it, then it is revived with its prior conversation context intact and processes the new call.
- AC-3: Given a follow-up call, when it completes, then its result is recorded and delivered exactly like a first call — the call records are siblings, both listable.

Edge cases:

- EC-1: Target identity does not exist → typed error distinguishing "never existed" from "expired past TTL". (Invalid input)
- EC-2: Target exists but is not in my lineage and no grant covers it → typed permission error; nothing is delivered. (Permissions)
- EC-3: Follow-up to a child that is mid-turn on the previous call → the new call queues and is delivered at the child's next boundary; both calls settle independently. (Concurrency)
- EC-4: Follow-up arrives while the child is being reaped → exactly one deterministic outcome: delivered before the reap, or the typed expired error; never a silent drop and never delivered-into-void. (State transitions)

### US-003: Fan out a batch of tasks in one call

**As a** Parent Agent, **I want** to pass a batch of tasks in one call invocation, **so that** parallel delegation costs one tool call and returns one consolidated view.

Acceptance criteria:

- AC-1: Given a batch of tasks (each with its own prompt, target, and optional result contract), when I invoke the call verb once, then each task becomes its own call record with its own child and identity.
- AC-2: Given the batch was accepted, when I inspect the response, then I see every call identity with its per-task acceptance state — accepted items proceed even when other items were rejected, and each rejection carries its own typed reason.
- AC-3: Given running batch items, when items complete at different times, then each completion is delivered independently; I never wait for the whole batch to hear about one item.
- AC-4: Given the configured concurrency budget, when the batch exceeds what can run at once, then excess items queue transparently — visible as queued, never silently dropped.

Edge cases:

- EC-1: Batch larger than the configured maximum → the whole batch is rejected with a typed error naming the cap; never silent truncation to the first N. (Limits)
- EC-2: Empty batch → typed error. (Empty / missing)
- EC-3: One item's target agent is unknown → that item rejects with its reason; sibling items proceed. (Invalid input)
- EC-4: Batch that would exceed the caller's child cap partway → items beyond the cap reject with the cap reason; accepted items stand. (Limits)
- EC-5: 100× typical batch volume → rejection stays fast and the error stays readable; no timeout from validation itself. (Scale)

### US-004: Retry a call safely with caller-chosen idempotency

**As an** Automation Integrator, **I want** to supply my own idempotency key on a call, **so that** network retries and crash-loops never double-delegate or burn the child budget.

Acceptance criteria:

- AC-1: Given a call with an idempotency key, when I submit the same key again (same caller), then I receive the original call's identity and current state instead of a new call — no second child, no second charge.
- AC-2: Given a replayed call, when I inspect the response, then it is explicitly marked as a replay.
- AC-3: Given the original call already completed, when I replay the key, then I receive the completed state with the result reference immediately.

Edge cases:

- EC-1: Same key, different payload → typed conflict error; the original is not mutated and no new call starts. (Invalid input)
- EC-2: Replay after the daemon restarted → same replay semantics; idempotency survives restarts. (Interruption)
- EC-3: Two concurrent submissions of the same key → exactly one call is created; the other receives the replay response. (Concurrency)
- EC-4: Key reuse after the original call's records were pruned by retention → treated as a new call; the response never claims replay. (State transitions)
- EC-5: Same key with a different result budget, overflow, or deadline → typed conflict — those values are part of the call's payload identity. (Repetition)

### US-005: Cancel an in-flight call and see it actually stop

**As a** Parent Agent, **I want** to cancel a call I made, **so that** abandoned work stops consuming budget and the record shows who stopped it and why.

Acceptance criteria:

- AC-1: Given an in-flight call, when I cancel it, then the child's execution for that call is actually stopped (not merely marked), and the call reaches a canceled terminal state carrying the actor and reason.
- AC-2: Given a canceled call, when the child's late result arrives anyway, then it is recorded as superseded evidence — visible for forensics, never flipping the terminal state.
- AC-3: Given a call with a declared deadline, when the deadline passes without completion, then the call terminates with a typed timeout state, distinguished from cancellation.

Edge cases:

- EC-1: Cancel a call that already completed → typed no-op response naming the terminal state; nothing changes. (State transitions)
- EC-2: Cancel during the child's terminal write → exactly one outcome wins (completed or canceled); the loser is recorded as superseded, deterministically. (Concurrency)
- EC-3: Cancel a queued (not yet started) batch item → it terminates canceled without ever starting a child turn. (Ordering)
- EC-4: Repeated cancel → idempotent; same terminal state, no error. (Repetition)
- EC-5: Child ignores the stop request → escalation follows the existing managed-stop path; the call still terminates with a typed reason. (Interruption)
- EC-6: Deadline declared as zero, negative, or non-integer seconds → typed invalid-deadline error at call time, on every surface. (Invalid input)

### US-006: Await a call result explicitly, bounded and resumable

**As a** Parent Agent, **I want** an explicit await on a call identity with a bounded timeout, **so that** when I need the result inline I can wait deliberately — and a timeout is a checkpoint, not a failure.

Acceptance criteria:

- AC-1: Given an in-flight call, when I await it with a timeout, then I receive either the typed result (call terminal) or a typed timeout outcome with a resume token — never a hang.
- AC-2: Given a timeout outcome, when I await again with the resume token, then no completion that happened in between is lost.
- AC-3: Given a call that is already terminal, when I await it, then I receive the terminal outcome immediately.
- AC-4: Given the await, when the target call is canceled or its child dies, then the await resolves promptly with that terminal outcome — never waits out the clock on a dead call.

Edge cases:

- EC-1: Await an unknown call identity → typed error immediately. (Invalid input)
- EC-2: Timeout above the allowed maximum → clamped or rejected per the documented bound; never silently unbounded. (Limits)
- EC-3: Daemon restart severs a client await → re-awaiting after reconnect converges on the durable call state with no lost completion. (Interruption)
- EC-4: Many concurrent awaits on one call → all resolve with the same outcome. (Concurrency)

## Typed Return

### US-007: Return a typed result as the terminal act

**As a** Child Agent, **I want** a native return surface that ends my assignment and carries my structured result in one act, **so that** "I finished" and "here is the answer" can never separate.

Acceptance criteria:

- AC-1: Given an active call with a result contract, when I invoke the return surface with a conforming payload, then the result is validated at admission, recorded durably in the same act that settles the call, and the call becomes completed.
- AC-2: Given my payload, when it is recorded, then it carries provenance — which agent produced it, for which call, when — and the stored payload is never truncated (previews are bounded; storage is whole).
- AC-3: Given completion, when the parent is notified, then the notification carries the typed result reference (US-017) — completing and delivering are one fact.
- AC-4: Given a call without a result contract, when I finish my turn normally, then my prose answer settles the call as its uncontracted result.

Edge cases:

- EC-1: Return payload valid against the contract but wrapped in a single extra envelope key → unwrapped and accepted; the repair budget is not spent on a formatting artifact. (Invalid input)
- EC-2: Return invoked twice for one call → first settles; second receives a typed already-settled error; the recorded result never mutates. (Repetition)
- EC-3: Return for a call that was canceled meanwhile → recorded as superseded evidence, terminal state unchanged (US-005 AC-2). (State transitions)
- EC-4: Return races the child's own crash → either the result settled durably first (call completed) or it didn't (call fails with the crash reason); no half-state. (Concurrency)
- EC-5: Return with no active call bound to me → typed error naming the expected binding. (Ordering)

### US-008: Get exactly one repair round on an invalid result

**As a** Child Agent, **I want** one repair prompt carrying the validation errors verbatim when my result fails the contract, **so that** honest mistakes get fixed without retry loops that degrade the answer.

Acceptance criteria:

- AC-1: Given an invalid return payload, when validation fails, then I receive the validator's errors verbatim in a repair prompt (bounded count), without the full contract re-pasted, and I get exactly one more attempt.
- AC-2: Given the second attempt also fails, when the call settles, then it ends in a typed invalid-result terminal state carrying both attempts' errors — never a fake success.
- AC-3: Given infrastructure failure during a return (provider retraction, transport loss), when the attempt is classified, then it does not consume the repair budget — infrastructure causes and validation causes are recorded as distinct.

Edge cases:

- EC-1: Contract itself fails to load/compile at return time → the call settles `failed` with a contract-fault reason attributing the fault to the contract, not the child; the repair budget is untouched. (Invalid input)
- EC-2: Repair prompt delivered but child returns nothing → call settles invalid-result on the child's turn end; never hangs open. (Empty / missing)
- EC-3: Validation errors overflow the bounded count → deterministic truncation of the error list, stated in the payload ("N more"). (Limits)

### US-009: Distinguish "completed without payload" from success and failure

**As a** Parent Agent, **I want** a distinct terminal state when a contracted child finishes without producing any result, **so that** I never treat silence as success or as a validation failure.

Acceptance criteria:

- AC-1: Given a call with a result contract, when the child's work ends without any return act (turn ended, no payload, no extractable result), then the call terminates in a completed-without-result state — distinct from completed, invalid-result, failed, canceled, and timeout.
- AC-2: Given that state, when I read the call, then it carries the child's final prose (bounded preview) so I can decide to re-call or escalate.

Edge cases:

- EC-1: Child ends its turn with unrelated prose only → completed-without-result (after the extraction fallback of US-011 finds no candidate). (Empty / missing)
- EC-2: Child stopped by TTL mid-work → failed with the TTL reason, not completed-without-result; cause taxonomy stays honest. (State transitions)

### US-010: Declare one result budget with an explicit overflow strategy

**As an** Agent Author, **I want** a single declared size budget per result contract with a stated overflow behavior, **so that** a large valid answer is never validated first and destroyed later by a second, conflicting ceiling.

Acceptance criteria:

- AC-1: Given a result contract, when I declare its budget (or accept the default), then exactly one ceiling governs that result end to end — there is no second hidden limit that can reject what validation accepted.
- AC-2: Given an over-budget result, when the overflow strategy is applied, then the outcome is the declared one — stored-with-bounded-projection or typed rejection telling the child the limit — and the strategy is visible on the call record.
- AC-3: Given any stored result, when surfaces render it, then projections are bounded previews with an explicit fetch path to the whole payload; the stored bytes are never truncated.
- AC-4: Given a single call or task, when I declare a per-call budget/overflow override, then it is capped by the configured maximum, snapshotted immutably on that call/run at admission, and later configuration changes never alter it.

Edge cases:

- EC-1: Budget declared above the platform maximum → rejected at declaration time with the allowed range. (Limits)
- EC-2: Result exactly at the boundary → accepted; boundary semantics documented and deterministic. (Limits)
- EC-3: Uncontracted (prose) result over the default budget → same overflow strategy applies; prose is not a bypass. (Ordering)

### US-011: Recover a result when the child answered only in prose

**As a** Parent Agent, **I want** the runtime to extract a contract-conforming result from the child's final text when the child failed to use the return surface, **so that** provider quirks don't turn correct answers into failures.

Acceptance criteria:

- AC-1: Given a contracted call whose child ended without the return act, when the child's final text contains a parseable candidate, then the newest valid candidate is admitted through the same validation gate and the call completes — marked with extraction provenance (result recovered from text, not returned).
- AC-2: Given extraction provenance, when I read the result, then I can tell it was extracted — the two paths are never conflated.

Edge cases:

- EC-1: Multiple candidate payloads in the text → newest valid one wins, deterministically. (Ordering)
- EC-2: Candidate parses but fails the contract → the US-008 repair round applies if the child is still live; otherwise invalid-result. (Invalid input)
- EC-3: Extraction disabled for the call (strict mode) → completed-without-result directly; strictness is the author's choice. (Permissions)

## Mailbox

### US-012: Message the parent mid-flight

**As a** Child Agent, **I want** to send my parent a message while I work (progress, a blocker, a question), **so that** coordination doesn't wait for completion.

Acceptance criteria:

- AC-1: Given my parent edge exists, when I send a message, then it is durably recorded before any delivery signal, and I receive a receipt describing what happened (delivered into a turn, woke the recipient, queued, or failed with reason).
- AC-2: Given the mailbox, when I use it, then no setup was ever required — the channel exists because the lineage edge exists; there is nothing to create, join, or configure.
- AC-3: Given my message, when the parent reads it, then it is stamped as coming from me (agent, session, call context) — never mistakable for operator input.

Edge cases:

- EC-1: Message to a parent that is past its own lifecycle (stopped, gone) → queued durably or failed with a typed reason per lifecycle state; the receipt says which. (State transitions)
- EC-2: Message larger than the message budget → sender-side typed rejection; nothing partial is delivered. (Limits)
- EC-3: Daemon restart between commit and delivery → the message survives and is delivered on recovery; the receipt trail shows the transition. (Interruption)
- EC-4: Child sends during its own terminal write → message and terminal result are ordered deterministically; the parent never sees the message attributed to the wrong call state. (Concurrency)

### US-013: Steer a running child without interrupting it

**As a** Parent Agent, **I want** to send course-corrections to a running child, **so that** I can redirect work without killing and re-spawning.

Acceptance criteria:

- AC-1: Given a running child, when I send it a message, then delivery happens at the child's next boundary (between tool calls, or as a new turn when idle) — a running tool is never interrupted mid-flight.
- AC-2: Given a parked child, when I message it, then the message revives it (ADR-003) and starts a turn carrying the message.
- AC-3: Given my steer, when the child was in the middle of settling the call, then a steer that arrived too late is reported on the completion record as missed — visible, never silently dropped.

Edge cases:

- EC-1: Steer to a child awaiting a human decision (blocked) → typed blocked outcome telling me to use the decision surface; the message does not answer prompts. (Permissions)
- EC-2: Multiple steers queued while the child runs one long tool → delivered in order at the next boundary; order is stable. (Ordering)
- EC-3: Steer to a child of another parent → denied by lineage rules unless a grant exists. (Permissions)

### US-014: See delivery receipts and outcomes for sent messages

**As a** Parent Agent, **I want** every send to tell me what happened and to keep telling me (held → delivered/failed), **so that** I can decide whether to wait, retry, or escalate.

Acceptance criteria:

- AC-1: Given any send, when it is accepted, then the receipt names the transport outcome from a closed vocabulary (delivered-into-turn, woke, queued, failed) — transport truth, not a guess about what the recipient will do.
- AC-2: Given a queued message, when its state later changes (delivered, expired, dropped by cap), then the sender can observe the updated outcome on the message record.

Edge cases:

- EC-1: Receipt requested for an unknown message id → typed error. (Invalid input)
- EC-2: Recipient exists but delivery keeps failing → after the documented policy, the message reaches a terminal failed state with reason; never an infinite queue. (Limits)

### US-015: Runaway message loops stop on their own

**As an** Operator, **I want** protocol-level brakes on messaging, **so that** two agents left alone cannot ping-pong forever on my budget.

Acceptance criteria:

- AC-1: Given a sender exceeding the per-sender rate limit, when it keeps sending, then sends are rejected with a typed rate-limit error naming the window.
- AC-2: Given identical repeated messages inside the dedup window, when they arrive, then duplicates are dropped with a receipt saying so.
- AC-3: Given a recipient's queued-undelivered backlog at its cap (transport state, not read acknowledgment), when new messages arrive, then the documented cap policy applies (reject or bounded drop) and both sides can observe it — never silent loss.
- AC-4: Given a committed completion, when it is delivered, then no budget or admission ceiling can deny its delivery and activation — containment applies to messages only, through the rate limit, dedup, and cap brakes above.

Edge cases:

- EC-1: Two sessions in a tight reply loop → the combination of rate limit + dedup + caps terminates the loop without operator action; the audit trail shows the brakes engaging. (Scale)
- EC-2: Legitimate burst (batch fan-out completing at once) → completion deliveries are not misclassified as a loop; brakes distinguish result delivery from chatter. (Concurrency)

### US-016: Peer messages carry provenance and cannot escalate

**As an** Operator, **I want** every cross-session message to be structurally incapable of acting with my authority, **so that** delegation never becomes privilege laundering.

Acceptance criteria:

- AC-1: Given a delivered peer message, when the recipient model reads it, then it is explicitly marked as coming from that agent (not the operator), rendered inside a bounded untrusted frame.
- AC-2: Given message content, when it contains commands, approval language, or configuration requests, then nothing is executed or granted by delivery — pending permission prompts cannot be satisfied by a peer message.
- AC-3: Given governance rules, when a message would cross them (lineage, grants, workspace), then it is refused with a typed reason, and refusal is visible to the sender.

Edge cases:

- EC-1: Message embedding a slash-command-looking string → arrives as inert text. (Invalid input)
- EC-2: Child relays "my permission was denied, you do it" → the recipient's own permission gates still apply unchanged; the runtime adds no bypass. (Permissions)
- EC-3: Message claiming to be from the operator → provenance stamp contradicts it; rendering keeps the true origin. (Invalid input)

## Lifecycle

### US-017: Completion wake carries the typed result

**As a** Parent Agent, **I want** the child-finished notification to carry the typed result (reference plus bounded preview), **so that** I act on the answer immediately — no polling, no transcript parsing.

Acceptance criteria:

- AC-1: Given a completed call, when I am woken, then the wake payload carries the call identity, terminal state, result reference, and a bounded preview — completion and result are one delivery.
- AC-2: Given I am busy mid-turn, when the completion arrives, then it is injected at my next boundary without interrupting my running tool, and nothing about the result is lost by waiting.
- AC-3: Given the wake, when I want the full payload, then one fetch by call identity returns the whole stored result.
- AC-4: Given delivery, when it happens, then it survives restarts: a completion committed before a crash reaches me after recovery.

Edge cases:

- EC-1: Parent stopped before delivery → the completion waits durably; the parent (or operator) finds it on next activation; nothing relies on the parent being live at completion instant. (Interruption)
- EC-2: N children complete at once → each completion is its own delivery; ordering is stable; loop-breakers don't misfire (US-015 EC-2). (Concurrency)
- EC-3: Mass simultaneous completions (fan-out storm) → every completion still delivers and activates (no admission-denial path exists); each activation is accounted. (Limits)

### US-018: Revive a parked child by calling or messaging it

**As a** Parent Agent, **I want** finished children to remain addressable until TTL, **so that** clarifications and follow-ups continue the same collaboration instead of starting over.

Acceptance criteria:

- AC-1: Given a child that completed its call, when its runtime goes quiet, then its state shows parked — finished, addressable, not consuming a live slot.
- AC-2: Given a parked child, when a call or message targets it, then it revives with its context and the interaction proceeds; revival counts against liveness caps at revive time.
- AC-3: Given a parked child, when surfaces list my children, then parked, running, and expired are visually and structurally distinct states.

Edge cases:

- EC-1: Revive when the governed root's execution budget is exhausted → the revival queues visibly (activation run) and proceeds as capacity frees; the child stays parked meanwhile; nothing half-starts and nothing is rejected. (Limits)
- EC-2: Two revival triggers race → exactly one revival; both interactions are delivered to the revived child in order. (Concurrency)
- EC-3: Revive a child whose agent definition changed since spawn → the child keeps its original bound definition; drift is surfaced, not silently absorbed. (State transitions)

### US-019: Calls past the child's TTL fail with a typed error

**As a** Parent Agent, **I want** addressing an expired child to fail fast and clearly, **so that** I know to spawn fresh instead of waiting on a ghost.

Acceptance criteria:

- AC-1: Given a child past TTL (reaped), when I call or message it, then I receive a typed expired error carrying the expiry time — distinguishable from "never existed" (US-002 EC-1).
- AC-2: Given expiry, when it happens with queued-undelivered messages or an unresolved call, then those reach terminal states with the expiry reason; the sender can observe them.

Edge cases:

- EC-1: Call arrives in the reap window (child being torn down) → deterministic single outcome: delivered-before-reap or expired-error; never delivered-into-void. (Concurrency)
- EC-2: TTL extended by config mid-life → documented policy applies (existing children keep their spawn-time TTL); behavior is stated, not accidental. (State transitions)

### US-020: Parent termination resolves outstanding calls predictably

**As an** Operator, **I want** a terminating parent to deterministically settle its outstanding delegation tree, **so that** no orphan children burn budget and no call stays open forever.

Acceptance criteria:

- AC-1: Given a parent with in-flight calls, when the parent reaches a terminal state, then each outstanding call resolves per the documented policy (drain: children stopped, calls terminally closed with the parent-terminal reason), following the descendant-drain precedent.
- AC-2: Given the drain, when I audit afterward, then every closed call names the cause; results that completed before the drain remain recorded and fetchable.

Edge cases:

- EC-1: Child completes concurrently with the parent's terminal transition → exactly one wins per record; a completed result is never destroyed by the drain. (Concurrency)
- EC-2: Parent crash (not clean stop) → recovery applies the same drain policy; restart-lossy limbo does not exist. (Interruption)
- EC-3: Grandchildren at depth 2-3 → the drain walks the whole governed subtree with cycle protection. (Scale)

## Subagents

### US-021: Describe an agent so callers can discover it

**As an** Agent Author, **I want** a short description on my agent definition, **so that** calling agents can pick the right specialist without me hand-feeding rosters into prompts.

Acceptance criteria:

- AC-1: Given an agent definition, when I author it, then a description field is part of the definition (alongside name, model, tools), carried through create/read surfaces and the definition digest.
- AC-2: Given a definition without a description, when it loads, then it remains valid — the roster shows it with an empty description; upgrade pressure, not breakage.
- AC-3: Given workspace and global definitions with the same name, when the roster renders, then the winning (shadowing) definition's description is shown.

Edge cases:

- EC-1: Description over the length bound → rejected at authoring/validation with the bound stated. (Limits)
- EC-2: Description containing markup/injection attempts → rendered inert wherever injected. (Invalid input)

### US-022: Pick a subagent from the roster without a lookup call

**As a** Parent Agent, **I want** the available agent roster visible in the call surface itself, **so that** choosing a specialist costs zero extra turns.

Acceptance criteria:

- AC-1: Given available definitions, when the call surface is presented to me, then it carries the roster (name + description), shadow-aware for my workspace.
- AC-2: Given a definition is added, removed, or reshadowed, when the surface next renders, then the roster reflects it — stale rosters converge without daemon restart.
- AC-3: Given I name an unknown agent anyway, when the call rejects, then the error carries the current roster (the recovery path is in the failure).

Edge cases:

- EC-1: Very large roster → bounded rendering with a documented policy (count/length caps + how to see the rest); never an unbounded prompt blob. (Scale)
- EC-2: Zero definitions available → the surface says so explicitly and points to creating one; calls by name fail with the empty-roster context. (Empty / missing)

### US-023: Delegate recursively to depth 3, honestly walled

**As a** Parent Agent, **I want** my subagents to delegate further, up to the configured depth, **so that** orchestrator → worker → helper trees work — with an honest wall instead of confabulated capability.

Acceptance criteria:

- AC-1: Given default configuration, when delegation nests, then depth 3 works and depth 4 is denied with a typed depth error.
- AC-2: Given a child at the depth limit, when its toolset is assembled, then the call verb is absent — it cannot even attempt to nest.
- AC-3: Given any child, when its prompt context is assembled, then it states the literal remaining delegation depth.
- AC-4: Given the operator raises or lowers the limit, when new spawns happen, then the new limit applies; containment (child caps, root execution budget, idle TTL) stays in force at every depth.

Edge cases:

- EC-1: A calls B, B calls A's definition (cycle by name) → allowed only within depth/budget walls; governance counts it against the governed root like any node; loop-breakers cover message cycles. (Repetition)
- EC-2: Depth raised mid-flight → in-flight children keep their spawn-time budgets; the change applies forward. (State transitions)
- EC-3: Deep tree fan-out (3 levels × batch) → the per-parent wall rejects over-cap NEW spawns typed; admitted work over the root execution budget queues visibly — one deterministic outcome per limit. (Scale)

### US-024: Manage the agent roster through structured surfaces

**As an** Automation Integrator, **I want** roster listing with descriptions over CLI/HTTP with structured output, **so that** external tooling can render and manage the same truth the model sees.

Acceptance criteria:

- AC-1: Given definitions, when I list agents via CLI (structured output) or API, then each entry carries name, description, scope (global/workspace/shadowed), and digest — matching what roster injection renders.
- AC-2: Given a definition create/update through the existing surfaces, when it includes the description, then round-trip read returns it unchanged.

Edge cases:

- EC-1: Listing in a workspace with shadowing → shadowed-out definitions are distinguishable from active ones. (Ordering)
- EC-2: Concurrent update + list → list returns a consistent snapshot. (Concurrency)

## Unification

### US-025: Loop node outputs honor the declared contract at settle

**As an** Agent Author, **I want** a Loop node's declared output contract enforced when the node settles — through the same regime as calls, **so that** a succeeded node can never carry an output missing required keys.

Acceptance criteria:

- AC-1: Given a Loop node with a declared output contract, when its work settles, then the payload is re-validated at settle; an invalid payload settles the node as invalid-output, never succeeded.
- AC-2: Given the unified regime, when I compare a call result and a node output, then validation behavior, failure vocabulary, provenance, and preview semantics are the same.
- AC-3: Given a node whose declared contract exists (produces/output declarations), when any surface resolves "the declared contract" for lint, review, or amendment, then exactly one resolution rule answers — the same one everywhere.

Edge cases:

- EC-1: Node output valid at capture but the stored payload lost keys before settle → settle validation catches it; the node fails typed, with both facts recorded. (Concurrency)
- EC-2: Declared contract on a node class that produces no payload → authoring-time validation error, not a runtime surprise. (Invalid input)

### US-026: Tasks declare a result contract enforced at completion

**As an** Agent Author, **I want** a task to optionally declare what a valid result looks like, **so that** autonomy workers hand back machine-usable results — same contract regime as everything else.

Acceptance criteria:

- AC-1: Given a task with a declared result contract, when a worker completes its run with a result, then the result is validated at completion admission; an invalid result is a typed completion failure the worker can repair (one round, US-008 semantics), not a silent acceptance.
- AC-2: Given a task without a contract, when it completes, then behavior is unchanged (uncontracted result), and the record says so.
- AC-3: Given a contracted task result, when read from any surface, then it carries the same provenance/preview semantics as call results.

Edge cases:

- EC-1: Contract added to a task while a run is in flight → the in-flight run completes under its start-time contract state; the change applies to later runs. (State transitions)
- EC-2: Result valid but over budget → US-010 overflow strategy, identically. (Limits)

### US-027: One validation and budget regime across every return path

**As an** Agent Author, **I want** every structured return in the system — call results, node outputs, task results, decision payloads, tool outputs — to share one validation, size, redaction, and provenance regime, **so that** learning it once is learning it everywhere.

Acceptance criteria:

- AC-1: Given the same contract and payload, when validated on any return path, then the verdict and error rendering are identical.
- AC-2: Given size ceilings, when any return path stores a payload, then one declared budget regime governs (US-010) — the historical conflict where one layer accepts and another rejects the same payload is gone.
- AC-3: Given entity-reference annotations in contracts, when payloads are validated, then live-entity checking behaves identically on every path.
- AC-4: Given one contract used by many calls, when validation runs at message frequency, then validation cost stays flat as volume grows for that contract — an author load-testing a fan-out sees no per-call compilation penalty.

Edge cases:

- EC-1: Legacy sentinel/failure encodings in stored outputs → replaced by explicit kinds in the unified regime; readers never string-sniff to classify a value. (State transitions)
- EC-2: A payload containing secret-shaped values → field-level redaction on write with the contract still satisfied; whole-payload rejection only when redaction cannot preserve validity, with a typed reason. (Permissions)

## Observability & UI

### US-028: See the delegation tree, inboxes, and results in one app

**As an** Operator, **I want** a dedicated OS-shell app for agent collaboration — the delegation tree, each agent's inbox, call states, and typed results, **so that** multi-agent work is a first-class surface, not archaeology through transcripts.

Acceptance criteria:

- AC-1: Given active and recent delegations, when I open the app, then I see the tree (who called whom, per-call state, depth), live-updating, spanning running, parked, and terminal nodes.
- AC-2: Given a call node, when I select it, then I see the call detail: prompt, contract, state timeline, typed result (schema-aware rendering with bounded preview + full fetch), provenance, and cost.
- AC-3: Given mailboxes, when I inspect an agent, then I see its messages with direction, provenance, delivery outcome, and age — no read/seen state exists (nothing in the runtime models it).
- AC-4: Given the app, when I act (message an agent, cancel a call, revive a parked child, stop a subtree), then every control maps to a real runtime operation with the same semantics as the CLI/tool path — no invented controls.
- AC-5: Given the whole app, when data renders, then it is workspace-scoped; nothing leaks across workspaces.

Edge cases:

- EC-1: Empty state (no delegations yet) → explains the feature and points to creating agents/calls; never a blank pane. (Empty / missing)
- EC-2: Deep/wide trees (depth 3 × large fan-out) → tree stays navigable (collapse, focus); rendering stays bounded. (Scale)
- EC-3: SSE disconnect/reconnect → the app resyncs to daemon truth without phantom states. (Interruption)
- EC-4: A control targeting a node that just went terminal → typed stale-action feedback, view refreshes; no silent failure. (Concurrency)

### US-029: Session detail shows calls and why the session woke

**As an** Operator, **I want** the session detail to show that session's calls (made and received) and to explain each wake, **so that** "why did this agent act?" always has a visible answer.

Acceptance criteria:

- AC-1: Given a session with calls, when I open its detail, then a calls panel lists them with state, counterpart, contract presence, and result preview — linking into the dedicated app for the tree view.
- AC-2: Given a wake caused by a completion or message, when I inspect the timeline, then the wake shows its cause and payload provenance (which call, which sender).

Edge cases:

- EC-1: Session with hundreds of calls → paginated/bounded panel with counts; totals stay truthful. (Scale)
- EC-2: Call whose counterpart session was pruned by retention → the call record still renders with its identities; links degrade gracefully. (State transitions)

### US-030: Blocked and failed delegations surface as needs-you signals

**As an** Operator, **I want** delegation states that need me (blocked child, invalid result, silent completion, drained tree) to reach the shell's attention surfaces, **so that** I intervene from the signal, not from polling.

Acceptance criteria:

- AC-1: Given a delegation event that needs operator attention, when it occurs, then the OS-shell attention surfaces (dock badge, app indicator) reflect it under the existing needs-you grammar, and it clears when its cause resolves — no manual dismissal exists anywhere in the shell.
- AC-2: Given multiple attention causes in one tree, when I open the app, then I can navigate directly from signal to the exact node needing action.

Edge cases:

- EC-1: Attention cause auto-resolves (parent handled it) → the signal clears without operator action. (State transitions)
- EC-2: Signal storm from a failing fan-out → signals coalesce per tree; the dock never counts one incident as N unrelated alerts. (Scale)

### US-031: Delegation cost is visible and truthful

**As an** Operator, **I want** wake/turn costs from calls and messages accounted in the same usage surfaces as everything else, **so that** delegation cost is never invisible or double-booked.

Acceptance criteria:

- AC-1: Given wake-consuming deliveries (ADR-004), when I inspect usage, then call/message activations appear under the same owner-keyed accounting as other activations — one substrate, no parallel books.
- AC-2: Given missing provider usage data, when totals render, then absence is stated (unavailable), never fabricated as zero.

Edge cases:

- EC-1: Batch fan-out at depth 3 → per-owner accounting still attributes each activation correctly to its owner and root. (Scale)
- EC-2: Daemon restart between an activation and its delivery accounting → the durable dedupe fence keeps the activation accounted exactly once, never doubled. (Interruption)

## Governance & Security

### US-032: Call-spawned children get subset-only permissions

**As a** Parent Agent, **I want** the call's spawn governance to enforce subset-only permission narrowing, **so that** delegation can only ever shed authority, never gain it.

Acceptance criteria:

- AC-1: Given a call that spawns, when I specify the child's permission atoms (tools, skills, paths, and peers), then any atom absent from my own effective set rejects the call with the widening atoms named.
- AC-2: Given extension hooks on the call lifecycle, when a hook mutates the request (narrow, annotate, deny), then governance re-validates after the hook — hooks cannot bypass safety.
- AC-3: Given the recursion wall (ADR-008), when a child at depth N spawns, then the grandchild's governance derives from the child's (already narrowed) set — narrowing composes down the tree.

Edge cases:

- EC-1: Hook attempts to widen → the widened result is rejected exactly like a caller widening. (Permissions)
- EC-2: Caller's own permissions changed between calls → each call validates against the caller's current effective set. (State transitions)

### US-033: Cross-workspace delegation stays hard-denied

**As an** Operator, **I want** calls and messages to stay inside the workspace, **so that** delegation cannot become a cross-workspace data path.

Acceptance criteria:

- AC-1: Given a target in another workspace, when a call or message addresses it, then it is denied at the boundary with a typed error; nothing is delivered, spawned, or recorded as delivered — matching the existing cross-workspace access posture.
- AC-2: Given all list/read/stream surfaces for calls, messages, and rosters, when queried, then results are scoped to the caller's workspace; identity or cache tricks cannot surface another workspace's records.

Edge cases:

- EC-1: Child spawned in workspace A while a session with the same name exists in B → addressing resolves only within A; no ambiguity across boundaries. (Permissions)
- EC-2: Workspace deleted with parked children/open calls → records reach terminal states with the workspace-removal reason; nothing dangles addressable. (State transitions)

### US-034: Secrets never ride result or message payloads

**As an** Operator, **I want** every payload crossing sessions scanned for secret-shaped content, **so that** a well-meaning child echoing its environment cannot exfiltrate credentials.

Acceptance criteria:

- AC-1: Given a result or message payload containing secret-shaped values (claim tokens, API keys, auth material), when it is admitted, then field-level redaction applies on write while keeping the contract satisfied; hash forms remain allowed.
- AC-2: Given redaction that cannot preserve contract validity, when admission decides, then the payload is rejected with a typed reason telling the child what to remove — never stored raw.
- AC-3: Given any projection (wake preview, UI, CLI, logs, events), when payloads render, then redaction has already happened at storage; projections cannot resurrect raw secrets.

Edge cases:

- EC-1: Secret split across fields to evade scanning → bounded rendering + untrusted framing still contain it; scanning best-effort is documented as such. (Invalid input)
- EC-2: Contract that requires a field whose value is inherently secret-shaped → authoring-time guidance rejects the contract pattern with a stated alternative (reference, not value). (Invalid input)

## Extensibility

### US-036: Observe the call and message lifecycle through hooks

**As an** Extension Author, **I want** typed lifecycle hooks for calls and messages, **so that** my extension reacts to delegation events without polling or tailing logs.

Acceptance criteria:

- AC-1: Given a registered hook on the call family, when a call is created, settled, canceled, published, drained, or a message is sent/delivered, then my hook receives the matching typed event with sanitized payload data (no raw secrets, ever).
- AC-2: Given a spawn-governance hook that narrows or denies, when it mutates the request, then governance re-validates after my hook — I can narrow, never widen.

Edge cases:

- EC-1: My hook errors or hangs → the runtime transition completes anyway (lifecycle hooks are fail-open); the failure is logged, never blocking. (Interruption)
- EC-2: My hook receives an event for a workspace my extension has no consent for → it does not fire; scoping is enforced before dispatch. (Permissions)

### US-037: Read calls and messages through the consented Host API

**As an** Extension Author, **I want** read methods for calls, results, and messages under the consent model, **so that** extensions can build on delegation data safely.

Acceptance criteria:

- AC-1: Given granted consent for call reads, when my extension lists or fetches calls, results, and messages, then it receives the same workspace-scoped projections the public API serves.
- AC-2: Given the v1 surface, when my extension attempts any call/message mutation through the Host API, then no such method exists — observation only.

Edge cases:

- EC-1: Consent denied or ungranted → typed consent denial; zero data crosses. (Permissions)
- EC-2: Reads never cross the workspace scope of the granting context. (Permissions)

## Governance (operator identity)

### US-038: Operator calls converge on one stable workspace identity

**As an** Operator, **I want** my CLI/API/app-originated calls to share one durable caller identity per workspace, **so that** idempotency, attention delivery, and audit history never split across ghost identities.

Acceptance criteria:

- AC-1: Given my first call in a workspace from any surface, when it is admitted, then a single durable operator-caller identity exists — and every later operator call from any surface reuses it.
- AC-2: Given that identity, when agents list targets or the runtime counts liveness/reaps sessions, then it never appears — it cannot be called, messaged, revived, or reaped, and consumes no capacity.
- AC-3: Given any operator-originated call, when I audit it, then the record shows the operator as the authenticated actor, distinct from the caller identity.

Edge cases:

- EC-1: Two simultaneous first calls (CLI + API) → exactly one identity wins; both calls proceed under it with their own actor records. (Concurrency)
- EC-2: Workspace deletion → the identity and its binding cascade away with the workspace's records. (State transitions)

## Network Bridge

### US-035: Publish a completed call into a Network conversation

**As a** Parent Agent, **I want** to optionally publish a completed call's result as evidence into a Network conversation, **so that** group coordination can see outcomes — without the call path ever depending on Network.

Acceptance criteria:

- AC-1: Given a completed call and my active Network participation, when I publish it to a conversation, then a Network message carrying the result evidence appears in that conversation, attributed to me, under Network's own rules.
- AC-2: Given the bridge, when anything flows, then it flows one way only: no Network message ever creates, answers, or mutates a call.
- AC-3: Given the repositioning, when a Loop author needs "post to a channel and await the group's decision", then the channel-result path serves it — and 1:1 typed returns are calls, with authoring surfaces steering to the right one.

Edge cases:

- EC-1: Publish without Network participation → typed error pointing at participation requirements; the call itself is unaffected. (Permissions)
- EC-2: Publish any call that is not `completed` (running, or a resultless terminal like failed/canceled/timeout/invalid-result) → typed error; only completed calls with a valid result publish. (Ordering)
- EC-3: Result over Network's message ceiling → bounded evidence (preview + reference), never a silent oversized failure. (Limits)
- EC-4: Publish the same call to the same conversation twice → the recorded message id returns as a replay (no second post); a different conversation publishes anew. (Repetition)
