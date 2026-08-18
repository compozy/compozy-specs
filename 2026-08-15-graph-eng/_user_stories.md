# User Stories: Loop Graph Completion (graph-eng)

Canonical behavior catalog for the Loop graph-completion feature set: human interaction, routing, fan-out completion strategies, and operator time travel. Companion to `_spec.md`; consumed by `_spec.md` Part II (component mapping), `_uiux.md` (surface states), and `_tests.md` (coverage matrix).

## Personas

- **Loop author** — a human or agent writing loop definitions. Declares human interactions, routes, fan-out strategies, and reliability behavior. Reads compile-time diagnostics.
- **Operator** — a human supervising live and historical runs through the web UI and command line. Approves, answers, repairs, compares, re-executes, and forks runs.
- **Responder** — the person a human request targets (often the operator; sometimes a teammate arriving through a notification deep link), or an agent the author explicitly permitted to answer.
- **Agent operator** — an agent inspecting and operating loops through structured command surfaces, under capability gates.

## Story Index

| ID     | Feature Area      | Persona        | Story                                                        |
| ------ | ----------------- | -------------- | ------------------------------------------------------------ |
| US-001 | Human interaction | Loop author    | Declare a structured input request that parks the run        |
| US-002 | Human interaction | Loop author    | Declare edit-before-execute review on a proposed action      |
| US-003 | Human interaction | Loop author    | Declare respond-as-result so a human answer replaces the action |
| US-004 | Human interaction | Operator       | Amend the output of a parked node before resuming            |
| US-005 | Human interaction | Loop author    | Control who may respond, with agent opt-in per node          |
| US-006 | Human interaction | Loop author    | Declare reminder/escalation/expiry behavior per request      |
| US-007 | Human interaction | Responder      | See pending requests with context and answer them            |
| US-008 | Human interaction | Agent operator | Answer a permitted request through structured surfaces       |
| US-009 | Routing           | Loop author    | Declare a router that picks exactly one route, with default  |
| US-010 | Routing           | Loop author    | Let a gate verdict select a pre-declared route               |
| US-011 | Routing           | Operator       | See which route was taken and why                            |
| US-012 | Fan-out           | Loop author    | fail_fast: first failure cancels remaining branches          |
| US-013 | Fan-out           | Loop author    | best_effort: proceed at a declared quorum, with partial state |
| US-014 | Fan-out           | Loop author    | race: first success wins, the rest cancel                    |
| US-015 | Fan-out           | Loop author    | Write conditions over fan-out progress                       |
| US-016 | Fan-out           | Loop author    | Name iteration variables so nested fan-outs do not shadow    |
| US-017 | Fan-out           | Loop author    | Run collections wider than the old branch cap                |
| US-018 | Fan-out           | Operator       | See skipped/pruned/canceled branches with their causes       |
| US-019 | Time travel       | Operator       | Compare two generations of one run                           |
| US-020 | Time travel       | Operator       | Compare two runs (fork vs origin)                            |
| US-021 | Time travel       | Operator       | Re-execute from a healthy node in the same run               |
| US-022 | Time travel       | Operator       | Fork a new linked run from a historical generation           |
| US-023 | Time travel       | Agent operator | Invoke diff/rerun/fork under capability rules                |
| US-024 | Reliability       | Loop author    | A broken stop condition exits the loop instead of wedging it |
| US-025 | Reliability       | Loop author    | A broken routing condition becomes a routable failure        |
| US-026 | Reliability       | Loop author    | Gate revision budgets count per gate                         |

## Human interaction

### US-001: Declare a structured input request that parks the run

**As a** loop author, **I want** to declare a node that asks a person for structured input mid-run, **so that** the loop can incorporate human knowledge without ending the run and losing its place.

Acceptance criteria:

- AC-1: Given a loop with an input-request node, when execution reaches it, then the run parks durably at that node, the run status shows it needs a person, and downstream nodes do not start.
- AC-2: Given a parked request, when the responder submits an answer matching the node's declared answer shape, then the answer becomes the node's output, the run resumes, and downstream nodes can reference it like any other node output.
- AC-3: Given a declared answer shape, when the author compiles the loop, then references to fields of the future answer are validated at compile time like any other node output.

Edge cases:

- EC-1: Answer does not match the declared shape → rejected with a validation message naming the failing fields; the request stays pending; nothing resumes. (Invalid input)
- EC-2: Node declares no answer shape → compile-time error; the author must declare what a valid answer looks like. (Empty/missing)
- EC-3: Two responders submit at the same time → exactly one answer is admitted; the loser receives a clear already-answered outcome. (Concurrency)
- EC-4: Responder submits after the request expired → rejected with an expired outcome; the expiry route already taken is not undone. (State transitions)
- EC-5: The daemon restarts while the request is pending → the request survives and is still answerable; no answer is lost or duplicated. (Interruption)
- EC-6: The same answer is submitted twice (retry after success) → the second submission returns the already-answered outcome without changing the recorded answer. (Repetition)
- EC-7: An input request inside a fan-out branch → each branch parks and is answered independently; answering one branch does not resume its siblings. (Scale/Ordering)

### US-002: Declare edit-before-execute review on a proposed action

**As a** loop author, **I want** a review step where a person can modify a proposed action's arguments before it executes, **so that** near-miss proposals get corrected instead of rejected and regenerated.

Acceptance criteria:

- AC-1: Given a node under edit review, when execution proposes the action, then the run parks showing the proposed arguments to the responder.
- AC-2: Given a parked review, when the responder approves with edited arguments, then the action executes exactly once with the edited arguments and the record shows both the original and edited versions with the editor's identity.
- AC-3: Given a parked review, when the responder approves unchanged, then the action executes with the original arguments.
- AC-4: Given a parked review, when the responder rejects, then the action does not execute and the author-declared rejection outcome applies (route or failure), with the rejection note carried as feedback.

Edge cases:

- EC-1: Edited arguments violate the action's declared input shape → rejected with a validation message; the review stays pending with the original proposal intact. (Invalid input)
- EC-2: Responder edits, then a second responder approves the original concurrently → exactly one decision wins; the loser sees already-decided. (Concurrency)
- EC-3: Rejection with no note → allowed; the default rejection feedback applies. (Empty/missing)
- EC-4: The action would have expired mid-review (its own timeout) → review decisions still apply; execution timing restarts when approved, and the record shows the review pause did not count against the action's own clock. (Interruption/Ordering)

### US-003: Declare respond-as-result so a human answer replaces the action

**As a** loop author, **I want** a responder to be able to answer *in place of* an action, **so that** a person can short-circuit work the model would otherwise do (or cannot do).

Acceptance criteria:

- AC-1: Given a node that permits respond-as-result, when the responder submits a response, then the action does not execute and the response becomes the node's output, validated against the node's declared output shape.
- AC-2: Given a respond-as-result outcome, when downstream nodes read the node's output, then they cannot distinguish it structurally from an executed result, and the run record clearly attributes it to the responder.

Edge cases:

- EC-1: Response fails the output-shape validation → rejected with field-level errors; the pending state is unchanged. (Invalid input)
- EC-2: Respond-as-result on a node whose declared output shape is absent → compile-time error: the author must declare the shape a human answer must satisfy. (Empty/missing)
- EC-3: Responder tries respond-as-result where the author only allowed approve/edit → rejected with an allowed-decisions list. (Permissions)

### US-004: Amend the output of a parked node before resuming

**As an** operator, **I want** to correct the output of a paused or parked node before the run resumes, **so that** a bad intermediate result does not poison everything downstream.

Acceptance criteria:

- AC-1: Given a paused/parked node that already has a settled output and a declared output shape, when the operator submits an amended output that validates, then the corrected value becomes the node's effective output (the recorded original is never rewritten), the amendment is recorded append-only with actor identity, reason, and timestamp, and resuming consumes the corrected output.
- AC-2: Given an amended node, when the operator inspects run history, then both the original and amended outputs are visible with provenance — the original is never silently erased.
- AC-3: Given a bad output that downstream nodes already consumed, when the operator amends it, then downstream does not re-run by itself — re-executing consumers is the rerun verb's job, and the surfaces say so.

Edge cases:

- EC-1: Amend attempted on a running (not parked) node → rejected: only parked/paused nodes are amendable. (State transitions)
- EC-1b: Amend on a parked cell that has no output yet → rejected with a deterministic no-output outcome; answering/resuming is the right verb. (Empty/missing)
- EC-2: Amended output fails shape validation → rejected with field-level errors; the original output stands. (Invalid input)
- EC-3: Two amendments race → exactly one wins per attempt; the second must re-read and re-submit. (Concurrency)
- EC-4: Amend on a node with no declared output shape → rejected with a deterministic error explaining amendment requires a declared shape. (Empty/missing)
- EC-5: Amend, then the run is canceled before resume → amendment remains in history; nothing executes. (Interruption)

### US-005: Control who may respond, with agent opt-in per node

**As a** loop author, **I want** human responders by default and agent responders only where I explicitly allow them, **so that** delegation to agents is a deliberate choice per interaction, never an accident.

Acceptance criteria:

- AC-1: Given a human-interaction node with defaults, when an agent attempts to answer, then it is denied with a deterministic not-permitted error.
- AC-2: Given a node where the author opted in agent responders, when a permitted agent answers, then the answer is accepted under the same validation rules as a human answer, and the record shows the agent identity.
- AC-3: Given any human-interaction node, when the agent that started the run attempts to answer it, then it is denied (self-response), even if agent responders are opted in.

Edge cases:

- EC-1: Author opts in agent responders on a node, then a child agent spawned by the run's initiator answers → denied: self-response covers the initiating chain, not just the direct starter. (Permissions)
- EC-2: Agent without the required capability attempts to answer an opted-in node → denied by capability gate before any validation. (Permissions)
- EC-3: Opt-in present but no agent ever answers → escalation ladder proceeds exactly as for human-only nodes. (Empty/missing)

### US-006: Declare reminder/escalation/expiry behavior per request

**As a** loop author, **I want** each human request to carry its own reminder → escalation → expiry ladder with a declared expiry outcome, **so that** unanswered requests degrade predictably instead of hanging runs forever.

Acceptance criteria:

- AC-1: Given a request with a declared ladder, when the first window elapses without an answer, then the reminder/escalation steps fire in order and each step is visible in run history.
- AC-2: Given a declared expiry outcome, when the request expires, then the run follows exactly that outcome (alternate route or halt) and the record names expiry as the cause.
- AC-3: Given a request without an explicit ladder, when it parks, then the same defaults that govern approval escalation today apply.

Edge cases:

- EC-1: Answer arrives in the same instant as expiry → exactly one wins deterministically; the record shows which. (Concurrency)
- EC-2: Expiry outcome routes to an alternate path whose subgraph is itself skipped by an earlier decision → compile-time error: expiry routes must be reachable forward destinations. (Ordering/Invalid input)
- EC-3: Daemon down across the whole expiry window → on boot, the expiry fires exactly once; no double-processing. (Interruption/Repetition)
- EC-4: Ladder with zero-length windows → compile-time error: durations must be positive, matching existing duration rules. (Invalid input)

### US-007: See pending requests with context and answer them

**As a** responder, **I want** every pending request to show me what is being asked, with the context the author chose to include, **so that** I can answer correctly without opening the run internals.

Acceptance criteria:

- AC-1: Given pending requests across runs, when the responder lists them, then each shows the request kind (input / edit review / respond / approval), the asking loop and node, the author-provided context payload, and what a valid answer looks like.
- AC-2: Given a notification delivered through an external channel, when the responder follows its link, then they land directly on the pending request, ready to answer inside CompozyOS.
- AC-3: Given a pending request, when it is answered by someone else first, then the responder viewing it sees the resolved state instead of a stale answer form.

Edge cases:

- EC-1: Author-provided context exceeds display limits → truncated with an explicit truncation marker; the full payload remains accessible. (Limits)
- EC-2: Deep link followed after the run terminated → a resolved/terminated view with the outcome, never an answer form. (State transitions)
- EC-3: Multiple requests pending in one run (parallel branches) → each listed separately, answerable independently, each naming its branch. (Scale)
- EC-4: Request context contains secret-shaped values → redacted per the existing redaction rules before display or notification. (Permissions)

### US-008: Answer a permitted request through structured surfaces

**As an** agent operator, **I want** to list and answer permitted human requests through structured commands with machine-readable output, **so that** supervision workflows can run without a browser.

Acceptance criteria:

- AC-1: Given pending requests, when the agent lists them through the command surface, then the output is structured, complete (kind, payload, answer shape, deadlines), and stable across retries.
- AC-2: Given a permitted request, when the agent submits a valid answer, then the outcome (accepted / rejected with reasons / already-answered / expired) is deterministic and machine-readable.

Edge cases:

- EC-1: Malformed answer document → deterministic validation error naming the fields; exit state distinguishes validation failure from permission denial. (Invalid input)
- EC-2: Answer to a request that does not exist → deterministic not-found outcome. (Empty/missing)
- EC-3: The same answer command re-run after success → already-answered outcome, idempotent, nothing changes. (Repetition)

## Routing

### US-009: Declare a router that picks exactly one route, with default

**As a** loop author, **I want** to declare "after this point, continue down exactly one of these named routes based on the run's state", **so that** cheap cases take cheap paths and risky cases take heavy paths.

Acceptance criteria:

- AC-1: Given a router with named routes and conditions, when execution reaches it, then exactly one route continues and every other route's dominated subgraph is skipped.
- AC-2: Given no condition matches, when the router evaluates, then the declared default route continues — a no-match failure is impossible at runtime.
- AC-3: Given a router without a default route, when the author compiles, then compilation fails with an error naming the router and the missing default.
- AC-4: Given route destinations that are not forward destinations in the graph, when the author compiles, then compilation fails naming the offending route.

Edge cases:

- EC-1: Two conditions both true → the first declared match wins; declaration order is the tiebreak and the record shows which condition matched. (Ordering)
- EC-2: Condition references a field that does not exist → compile-time reference error, same as every other condition in the document. (Invalid input)
- EC-3: Router with a single route plus default → valid; useful as a guard. (Limits/degenerate)
- EC-4: A route targets a node also reachable from an unskipped path → that node still runs via the live path; skip propagation only removes nodes whose every path was skipped, matching existing skip rules. (State transitions)
- EC-5: Router condition breaks at runtime → routable authoring failure per US-025, never a wedge. (Interruption)

### US-010: Let a gate verdict select a pre-declared route

**As a** loop author, **I want** a gate's outcome — including judge and human outcomes — to choose among routes I declared, **so that** quality verdicts can steer the path, not just pass or block it.

Acceptance criteria:

- AC-1: Given a gate whose outcome mapping names a route, when that outcome occurs, then execution continues down that route and skips the others, exactly as a router would.
- AC-2: Given today's silently-ignored route selection on gates, when this ships, then a gate outcome mapped to a route visibly takes it — the no-op behavior is gone.
- AC-3: Given a gate route mapping naming an undeclared or backward destination, when the author compiles, then compilation fails naming the gate and the route.

Edge cases:

- EC-1: Verdict maps to a route while another mapping says halt → mappings are per-outcome; each outcome has exactly one action; conflicting duplicate mappings are a compile-time error. (Invalid input)
- EC-2: Human approval outcome mapped to a route → follows the same constrained-override rules approvals have today; combinations that would bypass approval semantics are compile-time errors. (Permissions/Ordering)
- EC-3: Gate inside a fan-out branch routing per item → the route decision applies within that branch's lane only. (Scale)

### US-011: See which route was taken and why

**As an** operator, **I want** run history to show each routing decision with its cause, **so that** I can audit why a run took the path it took.

Acceptance criteria:

- AC-1: Given a completed router or routing gate, when the operator inspects the run, then the record shows the selected route, the condition or verdict that selected it, and when.
- AC-2: Given skipped routes, when the operator inspects nodes, then skipped nodes are visibly "not taken" (with the deciding router/gate named), never rendered as failed or missing.

Edge cases:

- EC-1: Default route taken → the cause explicitly says default-after-no-match, not a matched condition. (Empty/missing)
- EC-2: Routing decision in a generation that was later re-run → each generation's decision is recorded independently; history never collapses them. (Repetition)

## Fan-out completion and progress

### US-012: fail_fast — first failure cancels remaining branches

**As a** loop author, **I want** a fan-out mode where the first definitive branch failure cancels the remaining branches, **so that** doomed batches stop burning time and tokens.

Acceptance criteria:

- AC-1: Given fail_fast, when a branch fails definitively (retries exhausted or non-retryable), then all in-flight and pending sibling branches are canceled, and the join takes the failure path immediately.
- AC-2: Given the cancellation, when the operator inspects branches, then canceled branches read as canceled-by-strategy with the triggering branch named — distinct from their own failure.
- AC-3: Given a retryable branch failure with retries remaining, when fail_fast evaluates, then it does not fire — only definitive failure triggers it.

Edge cases:

- EC-1: Two branches fail definitively at the same time → one is recorded as the trigger deterministically; both read as failed; the rest read canceled-by-strategy. (Concurrency)
- EC-2: The failure arrives when every other branch already succeeded → nothing to cancel; the join takes the failure path. (Ordering)
- EC-3: Cancellation races a branch finishing successfully → the branch's completed result stands; completed work is never rewritten to canceled. (Concurrency/State transitions)
- EC-4: Daemon restart mid-cancellation → cancellation completes after boot; no branch resumes running. (Interruption)

### US-013: best_effort — proceed at a declared quorum, with partial state

**As a** loop author, **I want** to declare "proceed when N% / N branches succeed" together with an explicit statement that missing branches are not mandatory coverage, **so that** large batches tolerate stragglers without lying about completeness.

Acceptance criteria:

- AC-1: Given best_effort with a threshold, when the threshold is met and some branches failed or are still pending, then the join completes in an explicit **partial** state, pending branches are canceled, and downstream conditions can read that the result is partial with its coverage numbers.
- AC-2: Given best_effort grammar, when the author omits the non-mandatory-coverage declaration, then compilation fails: quorum requires declaring that missing branches are acceptable.
- AC-3: Given a partial join, when the run reaches its end, then the run outcome preserves partiality — a partial result never presents as complete anywhere (history, status, downstream reads).
- AC-4: Given the threshold is not met after all branches settle, then the join takes the failure path with the coverage numbers recorded.

Edge cases:

- EC-1: Threshold of 100% → equivalent to wait_all; valid but flagged by compile-time hint as redundant. (Limits)
- EC-2: Threshold that cannot be expressed (0%, negative, >100%) → compile-time error. (Invalid input)
- EC-3: Threshold met by canceled-then-recovered ordering races → threshold evaluation is monotonic: once met and admitted, late failures do not retract the partial completion. (Concurrency)
- EC-4: Empty collection (zero branches) → follows existing empty-collection semantics; strategy is irrelevant and recorded as such. (Empty/missing)
- EC-5: Downstream node treats partial output as if complete → possible by author choice, but the partial flag is present in the namespace; documentation and lint hint push authors to check it when the join is best_effort. (State transitions)

### US-014: race — first success wins, the rest cancel

**As a** loop author, **I want** the first successful branch to win and the rest to cancel, **so that** competing approaches (different models, strategies) can race for one answer.

Acceptance criteria:

- AC-1: Given race, when the first branch succeeds, then its result is the join's result, and all other branches are canceled with canceled-by-strategy causes.
- AC-2: Given race, when every branch fails, then the join takes the failure path with all failures recorded.

Edge cases:

- EC-1: Two branches succeed simultaneously → one winner recorded deterministically; the other's completed result remains visible in history but is not the join result. (Concurrency)
- EC-2: The winning branch's output fails a downstream contract later → normal failure semantics; race does not resurrect canceled losers. (State transitions)
- EC-3: Race with a single branch → valid; degenerates to plain execution. (Limits)

### US-015: Write conditions over fan-out progress

**As a** loop author, **I want** conditions (gates, stop conditions, strategy thresholds) to read fan-out progress — totals, successes, failures, rates — **so that** "proceed at 80%", "abort at 30% failures" are one-line declarations.

Acceptance criteria:

- AC-1: Given the progress namespace, when a condition references totals/success/failure counts or rates for a fan-out, then values reflect settled branch states at evaluation time.
- AC-2: Given a reference to progress outside any fan-out scope, when the author compiles, then compilation fails with a scope error, matching how iteration variables behave today.

Edge cases:

- EC-1: Progress read while branches are mid-flight → counts are of settled branches; the total includes not-yet-settled so rates are computable and monotonic. (Concurrency)
- EC-2: Progress of an empty collection → total zero; rates defined as zero; conditions must not divide by zero implicitly. (Empty/missing)
- EC-3: The bare `progress.*` alias referenced outside its fan-out's body → compile-time scope error; the qualified `nodes.<fanout>.progress.*` form stays valid wherever that node is referenceable. (Ordering)

### US-016: Name iteration variables so nested fan-outs do not shadow

**As a** loop author, **I want** to name each fan-out's item and index, **so that** nested fan-outs can reference outer and inner items without shadowing.

Acceptance criteria:

- AC-1: Given nested fan-outs with declared names, when body nodes reference the outer and inner names, then each resolves to its own scope's value, validated at compile time.
- AC-2: Given no declared names, when the author compiles, then the current default names keep working unchanged.

Edge cases:

- EC-1: Two fan-outs in one nesting chain declare the same name → compile-time collision error. (Invalid input)
- EC-2: Declared name collides with a reserved namespace root → compile-time error naming the reservation. (Invalid input)
- EC-3: Inner fan-out references its own name outside its body → compile-time scope error. (Ordering)

### US-017: Run collections wider than the old branch cap

**As a** loop author, **I want** a fan-out over hundreds of branches to just work, **so that** wide workloads stop hitting an arbitrary width cliff.

Acceptance criteria:

- AC-1: Given a collection materializing more branches than the previous hard cap, when the run executes, then it proceeds with only a bounded window of branches active at once, and completes with correct join arithmetic.
- AC-2: Given any width, when the operator inspects the run, then progress shows settled/active/pending branch counts truthfully throughout.

Edge cases:

- EC-1: Concurrency window larger than the branch count → everything runs concurrently; no artificial batching artifacts. (Limits)
- EC-2: Strategy interactions at width (fail_fast at branch 3 of 500) → pending never-materialized branches cancel without ever starting; the record distinguishes never-started from canceled-in-flight. (Scale/State transitions)
- EC-3: Daemon restart mid-window → the window re-forms from durable state; no branch runs twice, none is lost. (Interruption)
- EC-4: Very large collections stress display → operator surfaces aggregate counts rather than rendering every branch row; the full set stays queryable. (Scale/Limits)

### US-018: See skipped/pruned/canceled branches with their causes

**As an** operator, **I want** every not-run node and branch to say why — route not taken, branch pruned, canceled by strategy, never materialized — **so that** absence in a run is always explained.

Acceptance criteria:

- AC-1: Given any skipped/pruned/canceled node or branch, when the operator inspects the run, then it shows an explicit cause naming the deciding node/strategy, distinct from failure.
- AC-2: Given run history events, when a branch is pruned or canceled by strategy, then a dedicated event records it with the deciding cause — silence is never the only evidence.

Edge cases:

- EC-1: A node skipped by two independent causes (route skip + strategy cancel window) → the first applicable cause is recorded; the record is single-cause and deterministic. (Ordering)
- EC-2: Pruned branches at high width → events aggregate safely (bounded payloads), while per-branch causes remain queryable. (Scale/Limits)

## Time travel

### US-019: Compare two generations of one run

**As an** operator, **I want** a side-by-side comparison of two generations — node outcomes, outputs, verdicts, route decisions — **so that** I can see exactly what changed between iterations.

Acceptance criteria:

- AC-1: Given a run with multiple generations, when the operator requests a diff of two of them, then the result lists per-node status changes, output changes (with content-level diff where shapes allow), verdict changes, and routing/strategy decisions that differ.
- AC-2: Given the diff, when nothing differs for a node, then it is summarized as unchanged rather than dumped.

Edge cases:

- EC-1: Diff of a generation against itself → valid, empty diff. (Repetition)
- EC-2: Diff including a generation that carried forward outputs (not re-run) → carried-forward nodes marked as carried, not changed. (State transitions)
- EC-3: Outputs too large for inline diff → summarized with sizes/hashes and a pointer to full content. (Limits)
- EC-4: Diff of generations from different graph shapes (impossible within a pinned run) → not applicable within a run; cross-run diff covers shape divergence (US-020). (Invalid input)

### US-020: Compare two runs (fork vs origin)

**As an** operator, **I want** to compare a fork against its origin (or any two runs of the same loop), **so that** I can see what the changed inputs changed.

Acceptance criteria:

- AC-1: Given two runs of the same loop, when the operator requests a diff, then it shows input differences, per-node outcome/output differences at each run's final (or chosen) generation, and terminal outcome differences.
- AC-2: Given a fork, when the operator views either run, then the lineage link to the other is visible from both sides.

Edge cases:

- EC-1: Runs pinned to different definition versions → the diff states the definition divergence up front and compares only nodes present in both. (State transitions)
- EC-2: Diff across loops (different loop names) → rejected with a deterministic error; diff is same-loop only. (Invalid input)
- EC-3: One run still executing → allowed; the diff labels the live run's side as of its latest settled generation. (Concurrency)

### US-021: Re-execute from a healthy node in the same run

**As an** operator, **I want** to re-run a run from a chosen healthy node onward, **so that** I can recover from bad outputs or environmental hiccups without replaying the whole run.

Acceptance criteria:

- AC-1: Given a run and a chosen node, when the operator triggers rerun-from-node, then a new generation opens whose rerun set is that node plus its transitive dependents, everything else carries forward, and the generation's origin names the operator action.
- AC-2: Given the rerun, when the operator inspects lineage, then the new generation's parent is the generation it re-ran from, and the operator identity is recorded.

Edge cases:

- EC-1: Rerun from a node that is parked/pending in the current generation → rejected: rerun targets settled nodes; parked cells have their own resume verbs. (State transitions)
- EC-2: Rerun while the run is mid-generation → rejected with a deterministic busy outcome; rerun applies to settled generations. (Concurrency)
- EC-3: Rerun from a node inside a fan-out branch → re-runs that branch's lane and dependents; sibling lanes carry forward. (Scale)
- EC-4: Repeated rerun with the same explicit request id → idempotent, the duplicate returns the already-created generation; without a request id each acknowledged request is a fresh operation, and rapid duplicates are absorbed while the previous generation is still in flight. (Repetition)
- EC-5: Rerun on a terminal run → allowed (that is its main use); the run leaves its terminal state through the normal reactivation rules with provenance. (State transitions)

### US-022: Fork a new linked run from a historical generation

**As an** operator, **I want** to start a new run seeded from a past generation of an existing run, optionally overriding declared inputs, **so that** I can test "what if" from a known point without touching the original.

Acceptance criteria:

- AC-1: Given a source run and generation, when the operator forks with input overrides, then a new run starts whose baseline history (what its nodes see as the previous/best results) is the chosen generation's outputs, whose inputs are the overridden set, whose first generation executes the full body, and whose lineage records the source run and generation — so an overridden input can never coexist with a stale result derived from the old input.
- AC-2: Given the fork, when either run is inspected, then the fork link is visible from both directions, and the source run is byte-for-byte unaffected.
- AC-3: Given input overrides, when they violate the loop's declared input types/defaults, then the fork is rejected with the same validation errors a fresh start would produce.

Edge cases:

- EC-1: Fork from a generation whose outputs reference content that was pruned/retained-out → rejected with a deterministic error naming the missing content; no partial fork starts. (Empty/missing)
- EC-2: Fork of a fork → allowed; lineage chains. (Repetition/Scale)
- EC-3: Fork while the source run is executing → allowed from any settled generation; the fork does not interact with the live run. (Concurrency)
- EC-4: Fork with no overrides → valid; a fresh full-body attempt from the same baseline; record shows inputs identical. (Empty/missing)
- EC-5: Individual node-output editing at fork time → not offered; the surface only accepts input overrides (point corrections belong to amend, US-004). (Invalid input)
- EC-6: Concurrency limits of the loop (single-run policies) → the fork respects the loop's declared concurrency policy exactly like a fresh start would. (Limits/Permissions)

### US-023: Invoke diff/rerun/fork under capability rules

**As an** agent operator, **I want** structured commands for diff, rerun, and fork gated by capability, **so that** agents can repair and explore pipelines — within explicit safety rules.

Acceptance criteria:

- AC-1: Given the capability, when an agent invokes diff/rerun/fork, then the outcomes and outputs are structured and deterministic, matching the operator surfaces in behavior.
- AC-2: Given a run the agent itself started that is still executing, when that agent attempts rerun or fork on it, then it is denied with a deterministic self-operation error.
- AC-3: Given a missing capability, when an agent invokes rerun or fork, then it is denied by the capability gate with a deterministic error; diff is a workspace-authorized read requiring no extra capability.

Edge cases:

- EC-1: Agent reruns a run it started after that run reached terminal → allowed; the restriction is only while executing. (State transitions)
- EC-2: Agent forks another agent's live run → allowed with capability (forks never disturb the source). (Permissions)
- EC-3: Structured output consumed under retries → identical request yields identical (idempotent) results per US-021 EC-4 semantics. (Repetition)

## Predicate and gate reliability

### US-024: A broken stop condition exits the loop instead of wedging it

**As a** loop author, **I want** a stop condition that fails to evaluate to end the loop cleanly with a diagnostic, **so that** a typo in a late-stage condition can never wedge a healthy run into endless succession.

Acceptance criteria:

- AC-1: Given a stop condition that errors at evaluation, when the generation completes, then the run exits through its normal completion path with a recorded diagnostic naming the condition and error, instead of scheduling another generation or blocking.
- AC-2: Given the fail-open default, when the author explicitly overrides the failure behavior (on the node for node conditions; via the stop condition's object form for the contract), then the override applies.

Edge cases:

- EC-1: Condition errors only on some generations (data-dependent) → the generation where it errored exits; earlier successful evaluations stand. (State transitions)
- EC-2: The removed legacy field (progress-fingerprint fields) present in an old definition → compile-time unknown-parameter error; the definition must drop it. (Invalid input)
- EC-3: Cost-limit exhaustion (expression too expensive) → treated as evaluation failure with the cost diagnostic attached; the near-limit warning is visible before it ever fails. (Limits)

### US-025: A broken routing condition becomes a routable failure

**As a** loop author, **I want** a routing/branch condition that fails to evaluate to become a normal, routable authoring failure on that node, **so that** I can handle it like any other failure — route it, absorb it, or let it escalate.

Acceptance criteria:

- AC-1: Given a routing condition that errors, when the node evaluates, then the node fails with an authoring-class failure carrying the condition diagnostic, and the author's error handling (route/absorb) applies normally.
- AC-2: Given predicate cost tracking, when any condition crosses the warning threshold, then the warning is observable in run diagnostics (not silently discarded).

Edge cases:

- EC-1: Routing condition failure inside a fan-out branch → fails that branch only; strategy semantics then apply. (Scale)
- EC-2: Failure on the router's own condition → default route is NOT taken on evaluation failure (default covers no-match, not broken conditions); the node fails per AC-1. (Ordering/State transitions)
- EC-3: Author overrides to fail-open on a routing condition → allowed explicitly; the record shows the override was exercised. (Permissions/Repetition)

### US-026: Gate revision budgets count per gate

**As a** loop author, **I want** each gate's revision budget to count that gate's own revise decisions, **so that** two gates in one loop stop sharing a hidden counter.

Acceptance criteria:

- AC-1: Given two gates each with a revision budget, when one gate routes revise repeatedly, then only that gate's counter advances, and the other gate's budget is untouched.
- AC-2: Given a gate reaching its budget, when it would revise again, then its declared exhaustion behavior applies and the record names the exhausted gate.

Edge cases:

- EC-1: A gate evaluated for multiple items (fan-out) → the budget semantics are defined per gate per lane, recorded distinctly. (Scale)
- EC-2: Revisions spanning operator rerun generations → operator-origin generations do not consume authored revision budgets; only the gate's own revise decisions do. (Ordering/State transitions)

## Edge-Case Sweep Notes

Every story above was probed against all ten classes (invalid input, empty/missing, limits, permissions, concurrency, interruption, repetition, ordering, state transitions, scale). Classes without a recorded EC for a story were probed and found covered by existing loop-wide behavior: durable parking and boot reconcile cover interruption for every parked state (US-001..US-008); capability gates and actor provenance cover permissions on every operator/agent verb (US-019..US-023); and the epoch-fenced single-writer cell model covers concurrency for every write path not explicitly listed.
