# User Stories: Loop & Task Legibility

Canonical behavior catalog for the loop/task legibility program. Companion to `_spec.md`; consumed by `_spec.md` Part II (component mapping), `_uiux.md` (surface states), and `_tests.md` (coverage matrix).

## Personas

- **Supervisor** — the person whose work the loops are doing (today a developer or technical operator; the default read must not require runtime literacy). Starts runs, checks in between other work, answers what the run needs, collects the result.
- **Operator** — the same human in expert mode, or a runtime developer: diagnoses stuck runs, inspects nodes/attempts/errors, intervenes per node, audits lifecycle integrity.
- **Agent** — an AI agent operating CompozyOS through structured surfaces (command-line and programmatic interfaces, native tools). First-class user: needs the same truth as the views, machine-readable, with deterministic errors.

## Story Index

| ID     | Feature Area          | Persona    | Story                                                        |
| ------ | --------------------- | ---------- | ------------------------------------------------------------ |
| US-001 | Tasks calm default    | Supervisor | Task listings show only work items by default                |
| US-002 | Tasks calm default    | Supervisor | Opt-in filter reveals loop records, distinguished and linked  |
| US-003 | Tasks calm default    | Supervisor | Dashboards/inbox aggregates count only work items             |
| US-004 | Tasks calm default    | Agent      | Structured task listing defaults calm; explicit filter opts in |
| US-005 | Run page — plain      | Supervisor | Absorb a run's status in seconds (briefing test)              |
| US-006 | Run page — plain      | Supervisor | Progress as steps and rounds                                  |
| US-007 | Run page — plain      | Supervisor | Needs-you is unmissable and actionable                        |
| US-008 | Run page — plain      | Supervisor | Outcome and produced artifacts lead terminal runs             |
| US-009 | Run page — plain      | Supervisor | Narrated, durable run story                                   |
| US-010 | Loops at a glance     | Supervisor | Runs roster answers "which runs need me" in plain words       |
| US-011 | Operator depth        | Operator   | Live run graph with per-node state                            |
| US-012 | Operator depth        | Operator   | Complete node roster, healthy nodes included                  |
| US-013 | Operator depth        | Operator   | Generation history with outcomes                              |
| US-014 | Operator depth        | Operator   | Node interventions from the new views                         |
| US-015 | Operator depth        | Operator   | Bidirectional node ↔ session ↔ record links                   |
| US-016 | Lifecycle integrity   | Operator   | Terminal runs never leave live execution records              |
| US-017 | Lifecycle integrity   | Operator   | Pre-existing orphans repaired automatically                   |
| US-018 | Lifecycle integrity   | Agent      | Reconciliation is auditable                                   |
| US-019 | Agent manageability   | Agent      | Structured reads for nodes, generations, and timeline         |
| US-020 | Agent manageability   | Agent      | Follow run events headlessly with resume                      |

## Tasks calm default

### US-001: Task listings show only work items by default

**As a** Supervisor, **I want** every task listing to show only human-facing work items by default, **so that** an active loop never drowns my work list in mechanical rows.

Acceptance criteria:

- AC-1: Given a loop run with many action nodes, generations, and fan-out items, when I open the Tasks list or board, then zero loop-origin execution records (coordinator or cells) render, on any page of results.
- AC-2: Given loop records are excluded, when a loop run is active, then my own work items render exactly as before — grouping, counts, and ordering unaffected by the hidden records.
- AC-3: Given the exclusion, when counts or group headers render, then they never include hidden loop records (no "0 of 18" headers).

Edge cases:

- EC-1: Loop cells are the most recent activity during an active run → still absent from page 1 and every later page (exclusion is data-source-owned, never page-local).
- EC-2: A workspace with only loop records and no work items → Tasks shows its true empty state, not a leak of mechanical rows.
- EC-3: Loop records from workspace A → never visible in workspace B's listings under any filter.
- EC-4: A task created manually by a person that merely mentions a loop in its title → always visible (classification is by provenance, never by name matching).

### US-002: Opt-in filter reveals loop records, distinguished and linked

**As a** Supervisor, **I want** a hidden-by-default filter that reveals loop execution records, **so that** exclusion never becomes erasure when I need to see the machinery.

Acceptance criteria:

- AC-1: Given the Tasks surface, when I enable the loop-records filter, then coordinator and cell records appear, each visually distinguished as loop-origin work with plain-words identity (loop name, round, step) — never a machine id as primary text.
- AC-2: Given a revealed loop record, when I activate it, then I land on that run's page in the Loops area.
- AC-3: Given the filter is off (default), when I reload or navigate back, then it remains off — revealing is an explicit act each context.

Edge cases:

- EC-1: Filter enabled with no loop records in the workspace → truthful empty message scoped to the filter, not a generic empty state.
- EC-2: A revealed record whose run has since been deleted by retention → record still renders with its provenance; the run link states the run is no longer available.
- EC-3: Records for a run in another workspace → never revealed regardless of filter state.

### US-003: Dashboards/inbox aggregates count only work items

**As a** Supervisor, **I want** the Tasks dashboard and inbox to aggregate only work items, **so that** loop mechanics never inflate my counts or bury my inbox.

Acceptance criteria:

- AC-1: Given active loop runs, when the dashboard renders status breakdowns and active-work summaries, then loop-origin records are excluded from every number.
- AC-2: Given a loop cell escalates (quarantine/attention), when I check the Tasks inbox, then the escalation is not there — it surfaces through the loop attention lane (attention bell and run page) instead.

Edge cases:

- EC-1: Every current escalation belongs to loops → Tasks inbox shows its empty state while the attention bell still shows the loop lane; nothing is lost.
- EC-2: A person-owned work item and a loop cell both need attention → inbox shows exactly the work item; the bell shows both lanes per its own contract.

### US-004: Structured task listing defaults calm; explicit filter opts in

**As an** Agent, **I want** the structured task-listing surfaces to apply the same calm default with an explicit include filter, **so that** the machine truth matches the human truth with no divergent contracts.

Acceptance criteria:

- AC-1: Given a default structured task-list call, when loop runs are active, then the response contains zero loop-origin records, on every listing surface with the same semantics.
- AC-2: Given the explicit include filter, when I request loop records, then each returned record carries structured loop provenance (run identity, step, round, item) as fields — no id-string parsing required.
- AC-3: Given an invalid filter value, when I call the listing, then I receive a deterministic, documented error.

Edge cases:

- EC-1: Drill-down by parent record id → returns that coordinator's cells even without the global filter (explicit parentage is already an explicit request).
- EC-2: Pagination cursor taken with the filter on, replayed with the filter off → deterministic, documented behavior (cursor binds its filter context; no silent mixing).
- EC-3: Existing consumers that assumed loop records in defaults → contract change is documented in the agent path; no compatibility shim exists.

## Run page — plain register

### US-005: Absorb a run's status in seconds (briefing test)

**As a** Supervisor, **I want** the run page's default read to answer what is running, what needs me, how far along it is, and what it produced, **so that** I understand my run in under 30 seconds without runtime literacy.

Acceptance criteria:

- AC-1: Given any run, when the page opens, then the default register shows: plain-language status line, needs-you (when any), step/round progress, and outcome/artifacts (when terminal) — with no machine ids as primary text anywhere in the default register.
- AC-2: Given a running run, when state changes (node finishes, request opens), then the default register updates live without reload.
- AC-3: Given the default register, when deeper mechanics exist (nodes, attempts, events), then they are reachable in one step (disclosure), and the default view never pays for them.

Edge cases:

- EC-1: Run opened before its first round starts (queued/admission) → status reads as plainly waiting to start, with the reason when one exists (concurrency limit), never a blank or spinner-only view.
- EC-2: Run in a watching/dormant state → reads as calmly idle with what it is waiting for; no fake activity.
- EC-3: Deep link to a run from another workspace → workspace scoping applies; no cross-workspace leak.

### US-006: Progress as steps and rounds

**As a** Supervisor, **I want** progress expressed as steps within the current round, **so that** a loop without fan-out (the common case) still shows a real completion signal.

Acceptance criteria:

- AC-1: Given a running round, when the progress renders, then it reads as settled action steps out of total action steps for that round (e.g., "step 3 of 6"), plus a round counter when past round 1.
- AC-2: Given fan-out on a step, when progress renders, then the step shows a derived rollup count (items done/total), never one element per item.
- AC-3: Given attempts on a step, when progress renders, then the attempt is step metadata ("attempt 3"), never a sibling step.

Edge cases:

- EC-1: Single-pass loop finishing on round 1 → progress completes without round noise ("round 1" is not shown).
- EC-2: Route-not-taken / never-materialized branches → excluded from the step totals and rendered neutrally (absence is calm).
- EC-3: A round where every action step is parked (waiting/paused) → progress states parked plainly with the dominant reason, not a frozen bar.
- EC-4: Control-only segments between action steps → contribute no steps; progress skips them without gaps in numbering.

### US-007: Needs-you is unmissable and actionable

**As a** Supervisor, **I want** anything awaiting me rendered first and actionable in place, **so that** I never miss an approval, request, or quarantine.

Acceptance criteria:

- AC-1: Given an open approval, request, or quarantine, when the page renders in any state of disclosure, then the needs-you element is visible (it never collapses) and leads the page.
- AC-2: Given a needs-you card, when I read it cold, then it states the action requested, which step/agent asked, and the choices — in plain words readable without context.
- AC-3: Given I act (approve, respond, requeue), when the action completes, then the card resolves in place and the story records the resolution.

Edge cases:

- EC-1: Multiple simultaneous needs-you → ordered list with a count; acting on one never hides the others.
- EC-2: The request is answered elsewhere (command line, another window) while I look at it → the card resolves live with who answered.
- EC-3: A request expires per its policy → the card states the expiry plainly and never auto-retries.

### US-008: Outcome and produced artifacts lead terminal runs

**As a** Supervisor, **I want** a finished run to lead with its outcome and what it produced, **so that** collecting results requires no forensics.

Acceptance criteria:

- AC-1: Given a terminal run, when the page opens, then the outcome reads in plain words (done, blocked, failed, exhausted, stalled, canceled, no-op) with its cause, followed by produced outputs/artifacts with links.
- AC-2: Given a failed run, when the default register renders, then the failure signal and plain-language cause are visible without disclosure — the failure never collapses.
- AC-3: Given partial results (some steps succeeded before a terminal failure), when outputs render, then they are labeled as partial/preliminary.

Edge cases:

- EC-1: Run produced no outputs (no-op) → states that plainly; no empty artifact section pretending content.
- EC-2: Canceled run → shows who canceled and when.
- EC-3: Output content unavailable (blob pruned) → the entry stays with a truthful "content no longer stored" note.

### US-009: Narrated, durable run story

**As a** Supervisor, **I want** the run story told as titled plain-language beats with full history preserved, **so that** returning after hours or reloading never loses the tail.

Acceptance criteria:

- AC-1: Given a run's story, when beats render, then each has a plain title in meaning terms ("second reviewer rejected the draft"), never mechanics ("event node_failed g2").
- AC-2: Given a reload or a return days later, when I read the story, then the full history from run start is reachable (paged), not just a recent window.
- AC-3: Given live progress, when new beats arrive, then they append in order with no duplicates or gaps.

Edge cases:

- EC-1: Very long run (thousands of beats) → paged reading stays responsive; oldest history loads on demand.
- EC-2: Multiple browser windows on the same run → both converge on the same story order.
- EC-3: Run forked/time-traveled → the story records the fork point and links the related run.

## Loops at a glance

### US-010: Runs roster answers "which runs need me" in plain words

**As a** Supervisor, **I want** the runs roster to lead with outcome/attention in plain words, **so that** scanning many runs takes seconds.

Acceptance criteria:

- AC-1: Given the runs roster, when rows render, then each leads with plain status/outcome and a needs-you marker when applicable; machine ids are secondary text at most.
- AC-2: Given a run needing me, when I scan the roster, then that run is visually distinct and reachable in one activation.

Edge cases:

- EC-1: No runs yet → empty state explains how to start one.
- EC-2: Dozens of active runs → roster remains scannable; needs-you runs never sort below terminal ones.

## Operator depth

### US-011: Live run graph with per-node state

**As an** Operator, **I want** the run's authored graph rendered with live per-node state, **so that** I can see the shape of execution — what runs, what waits, what failed — at a glance.

Acceptance criteria:

- AC-1: Given a running run, when I open the graph view (operator register), then every authored node renders with its current state (queued, running, succeeded, failed, retrying, waiting, paused, quarantined, canceled, not-taken) using color plus icon/text (never color alone).
- AC-2: Given fan-out on a node, when the graph renders, then the node shows a derived rollup (e.g., "7/10 · 1 failed"), never one element per item.
- AC-3: Given state changes, when the run progresses, then node states update live.
- AC-4: Given a node, when I select it, then I see its detail (attempts, timing, error class, session link, cell record link) without leaving the run page.

Edge cases:

- EC-1: Wide fan-out (100+ items) → rollup stays a count; the view never renders per-item elements.
- EC-2: Route-not-taken branches → neutral rendering (calm absence), visually distinct from failure.
- EC-3: Deep or wide graphs → layout stays readable (scroll/pan), and the current activity is locatable without hunting.
- EC-4: Terminal run → the graph renders the final state faithfully (not stuck on last-live frame).

### US-012: Complete node roster, healthy nodes included

**As an** Operator, **I want** a roster of every node × round with state, attempts, and timing — including healthy ones, **so that** "what is my run doing right now" always has an answer.

Acceptance criteria:

- AC-1: Given a run, when I open the roster, then every authored action node of every round appears with state, attempt count, next-retry time when scheduled, timing, and links — healthy queued/running/succeeded nodes included.
- AC-2: Given a node with multiple attempts, when I inspect it, then the attempt history is readable in order with each attempt's failure class/disposition.
- AC-3: Given rounds, when the roster renders, then round membership is explicit and filterable.

Edge cases:

- EC-1: Run with zero action nodes reached (terminal before round 1) → roster states that truthfully.
- EC-2: Node canceled by strategy → shows the strategy cause, distinct from operator cancel.
- EC-3: Roster of a wide fan-out node → items grouped under the node with a rollup and expandable detail, never a flat flood.

### US-013: Generation history with outcomes

**As an** Operator, **I want** per-round history with outcomes, scores, and verdicts, **so that** I can read how the run converged (or did not).

Acceptance criteria:

- AC-1: Given a run past round 1, when I open the history (operator register), then each round lists its outcome, score/verdict when the loop defines them, and node-level results.
- AC-2: Given two rounds, when I compare them, then the existing compare/fork entry points remain reachable from the history.

Edge cases:

- EC-1: Round interrupted by crash/restart → renders with its true partial state and the recovery note, not fabricated completion.
- EC-2: Loop without scoring → history shows outcomes without inventing score columns.

### US-014: Node interventions from the new views

**As an** Operator, **I want** the existing node verbs (pause, resume, cancel, kill, requeue, amend, approve, respond) reachable from the graph and roster, **so that** seeing a problem and acting on it are one motion.

Acceptance criteria:

- AC-1: Given a node in an actionable state, when I open its actions from graph or roster, then the applicable verbs for that state are offered — and only those.
- AC-2: Given an intervention, when it completes, then node state, roster, story, and progress all reflect it live.
- AC-3: Given an intervention on a node in a terminal run, when I attempt it, then it is rejected with a deterministic, plain explanation.

Edge cases:

- EC-1: Two operators act on the same node concurrently → single winner; the loser sees the new state, no double effect.
- EC-2: Intervention on a node whose state just changed (stale view) → rejected or re-validated against current state; never applied blindly.

### US-015: Bidirectional node ↔ session ↔ record links

**As an** Operator, **I want** every node linked to its session and execution record and back, **so that** navigation never dead-ends.

Acceptance criteria:

- AC-1: Given a node with a session, when I inspect it, then "open session" works during and after the run (not live-only).
- AC-2: Given a node's execution record (cell), when I view it anywhere, then a link back to its run page exists and works.
- AC-3: Given a child-loop node, when I inspect it, then the child run link works in both directions (child page names its parent).

Edge cases:

- EC-1: Session ended and pruned → link degrades to a truthful "session no longer available" instead of an error page.
- EC-2: Node that never started (not-taken) → no fabricated links; detail states it never materialized.

## Lifecycle integrity

### US-016: Terminal runs never leave live execution records

**As an** Operator, **I want** every terminal transition to settle the run's coordinator and cell records, **so that** finished/failed/canceled runs never leave live records behind.

Acceptance criteria:

- AC-1: Given a run reaching any terminal outcome by any path (natural completion, cancel, kill, crash recovery, child-stop), when I list execution records afterward, then the coordinator and every cell are settled — none live.
- AC-2: Given a daemon crash mid-run, when the daemon boots, then reconciliation settles any records whose run is terminal within one sweep cycle, without operator action.
- AC-3: Given an active (non-terminal) run, when reconciliation sweeps, then its live records are untouched.

Edge cases:

- EC-1: Crash exactly between run-terminal write and record settlement → sweep converges the records; no permanent divergence window beyond one cycle.
- EC-2: Sweep racing a live settlement of the same records → single winner, idempotent result, no state flip-flop.
- EC-3: Terminal run whose records were already settled → sweep is a no-op (idempotent), emitting nothing.

### US-017: Pre-existing orphans repaired automatically

**As an** Operator, **I want** the first boot with this fix to settle orphans that already exist, **so that** past crashes stop haunting listings forever.

Acceptance criteria:

- AC-1: Given pre-existing live coordinator/cell records of terminal runs (e.g., the two known QUEUED coordinators), when the daemon first boots with this change, then those records settle automatically with an audit reason.
- AC-2: Given repeated boots, when reconciliation re-runs, then repair is idempotent — no duplicate events, no re-settling.

Edge cases:

- EC-1: Orphan whose run row no longer exists (retention) → record settles with an explicit "run missing" audit reason rather than being skipped forever.
- EC-2: Large orphan backlog → every orphaned record loses work-eligibility before any work is picked up after boot (no stale terminal-run work can start); the remaining repair (audit events, provenance) may finish across the first cycles without blocking readiness.

### US-018: Reconciliation is auditable

**As an** Agent, **I want** sweep-settled records to carry a distinct audit reason, **so that** reconciliation is distinguishable from natural completion in any inspection.

Acceptance criteria:

- AC-1: Given a record settled by reconciliation, when its history is read on any surface, then the settlement event carries a reconciliation reason distinct from natural completion, including which terminal run state triggered it.
- AC-2: Given natural completion, when history is read, then no reconciliation reason appears.

Edge cases:

- EC-1: Reading a swept record's history on any surface → the reconciliation reason is a structured, machine-readable field on the event (distinguishable without prose parsing); no cross-record audit query is promised.

## Agent manageability

### US-019: Structured reads for nodes, generations, and timeline

**As an** Agent, **I want** structured reads for everything the run views show — node roster, rounds, timeline — **so that** I can supervise runs headlessly with the same truth.

Acceptance criteria:

- AC-1: Given a run, when I query its node roster, then I receive every node × round with state, attempts, timing, session/record identities — the same data the graph and roster views render.
- AC-2: Given a run, when I read its timeline, then pages are stable, ordered, gap-free, and resumable from a position token.
- AC-3: Given an unknown run or invalid position token, when I query, then I receive a deterministic, documented error.
- AC-4: Given a run, when I request its briefing, then I receive the same verdict the run page leads with — plain-meaning status, current blockers (what it waits on, since when), and the concrete unblock action — as structured output.

Edge cases:

- EC-1: Timeline read while events append → pagination remains stable (no skipped/duplicated entries across pages).
- EC-2: Query scoped to another workspace's run → denied by scoping, not an empty 200-style success.
- EC-3: Very large roster (wide fan-out × many rounds) → paged with derived rollups available so agents need not fetch every item.

### US-020: Follow run events headlessly with resume

**As an** Agent, **I want** to follow a run's events from the command line with resume, **so that** headless supervision does not require the web app.

Acceptance criteria:

- AC-1: Given a running run, when I follow its events, then I receive each new event as structured output as it happens, and the stream ends cleanly when the run reaches terminal.
- AC-2: Given a disconnect, when I resume from my last position, then I receive exactly the missed events — no gaps, no duplicates.
- AC-3: Given a terminal run, when I follow it, then I receive the remaining history from my position and a clean end, not a hang.

Edge cases:

- EC-1: Following a run with no events yet → waits quietly; first event arrives when produced.
- EC-2: A resume position beyond the run's history head → deterministic error naming the current head, never a silent hang; a paged-read continuation from another run or a fork → deterministic "different history" error, never spliced histories.
