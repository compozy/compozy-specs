# PRD: Loop Node Lifecycle & Failure Contract

Spec 1 of the loop-hardening train derived from the `.compozy/tasks/loop-ideas` corpus. Defines the complete contract for how a loop node classifies failure, reacts, retries, stays alive, pauses, cancels, waits, and recovers — and how operators and agents observe and repair all of it.

## Overview

A Loop is the deterministic runtime program CompozyOS owns and executes: goal → act → verify → stop, with a body of typed nodes, executed in generations with repair context, under caps and budgets. That outer machinery shipped with loops-paper-adoption. What is still missing is the **inner contract**: what a single node does when the world fails around it.

Today that contract barely exists. A loop author has no failure vocabulary — no error routes, no allow-fail, no per-node retry, no notifications, no time bounds. A transient network blip escalates a whole generation; two consecutive failures of one node kill the entire run; a run that ends is dead forever (no requeue, no repair-and-continue); a crashed agent session restarts days of work from zero; an approval nobody noticed waits invisibly forever; and one sick external dependency takes down every healthy lane of a run.

This spec gives every persona the missing half of the loop story:

- **Loop authors** (human and agent) get a small, declarative failure vocabulary — routes, absorption, retries, time bounds, reactions — with safe behavior when they declare nothing.
- **Operators** get live-repair verbs (pause, resume, requeue, cancel vs kill) and truthful inventories (waiting, quarantined, needs-attention) instead of run death and log archaeology.
- **Managing agents** get every new state and verb through structured surfaces with parity, so loops are supervisable and repairable with no UI.
- **The workload itself** — agents running for hours or days — gets a platform that never kills work for being slow, distinguishes "quietly working" from "dead", and recovers crashes by continuation instead of repetition.

The value is direct: fewer dead runs, cheaper failures (a blip costs a delay, not a generation; a crash costs minutes, not days), and failure handling that agents can author, observe, and operate end to end.

## Goals

What ships, stated as observable behavior:

- **A failure vocabulary authors can hold in their head.** A node's failure behavior is declarable in a few lines: retry policy, an error route, explicit absorption, reactions, optional time bounds. Every piece is optional; the undeclared default is always safe (escalation into generation repair, never silent loss).
- **Truthful failure classification.** A result body that declares failure is a failure, with the real message. Authoring mistakes (bad references) are distinguished from runtime failures and reported with what-is-available diagnostics. Cancellation, semantic rejection, and transport failure are different classes with different automatic behavior.
- **Cheap recovery for cheap failures.** Mechanical nodes retry transient failures automatically with backoff; retries are visible (attempt, next attempt, last failure) and always count against caps and budgets.
- **Expensive work is never re-executed blindly.** Agent and sub-loop nodes do not auto-retry. A confirmed-dead session resumes from its last progress checkpoint automatically, as continuation with provenance — bounded, then surfaced.
- **Nothing dies by clock.** No node has a default duration limit. Health is evidence of life (streamed activity, in-flight tool executions, transport presence), never elapsed time. True prolonged silence flags attention; it never auto-kills.
- **Nothing rots invisibly.** Everything waiting on a human or an event is enumerable with age and reason. Waits never expire by default; authored escalation ladders (re-notify → notify → route) fire when declared; expiry without a route surfaces as needing attention.
- **One sick dependency degrades one lane.** Target health is tracked per external target from transport-class failures only (classified as target-unavailable when the breaker is open); an open target fails its nodes fast with an explicit cause while healthy targets keep working. Repeat offenders quarantine with full repair context and requeue cleanly after repair.
- **Notification is a one-liner.** Reactions on node triggers and loop terminal outcomes emit events and run tools with pre-bound context and links — fail-open, observe-only, dispatched only after the state they announce is durable.
- **One event, one run.** Watch-event admission suppresses duplicates loudly and durably; a source without stable event identity fails validation instead of running unprotected.
- **Everything above is agent-operable.** Every new state, inventory, and verb exists on command-line, programmatic API, and native agent-tool surfaces with parity, structured output, and deterministic errors.

Success criteria (observable from outside the system):

- A loop definition with zero failure annotations behaves exactly as today's engine plus family-default retries — and its full failure story is explainable in one sentence.
- A pulled network cable during a mechanical node costs one backoff delay, not a generation; a killed agent process during a days-long node costs one resume from checkpoint, not a restart.
- An operator can pause a node, fix a credential, resume it, and watch the run finish — without the run ever leaving the live state.
- A quarantined node can be diagnosed entirely from its quarantine entry, requeued, and complete — while the run's independent branches kept working the whole time.
- Redelivering the same watch event N times yields exactly one run and N−1 visible suppressions.
- An agent, using only structured surfaces, can enumerate everything waiting or quarantined across a workspace and repair it.

### Agent/operator manageability outcome

Operators and agents must be able to inspect, configure, operate, and repair every capability in this contract outside the web UI: enumerate waiting/quarantined/flagged work, read effective family defaults and their sources, pause/resume/reset nodes, requeue quarantined work, cancel cooperatively or kill immediately, and observe every lifecycle transition as truthful ordered events. UI-only control of any of it is an incomplete delivery.

### Extension ecosystem expectation

Extension surfaces participate on both sides of the contract: extension-provided tools are invocable as reaction effects; extension-provided watch sources are subject to the same stable-event-identity requirement as built-in ones; and all lifecycle transitions introduced here (retry, pause, quarantine, liveness episodes, terminal outcomes) are observable through the existing lifecycle-event surfaces so extensions can build on them. No new extension kinds are required; existing extension points gain a richer, truthful signal stream.

## User Stories

Full catalog: [Full user stories](_user_stories.md). Index by area:

- **US-001..004 — Classification & diagnostics:** payload-declared failure, authoring-vs-runtime distinction, remediation hints into repair, predicate failure policy.
- **US-005..007 — Failure handling & routing:** error routes, explicit absorption, escalate-by-default.
- **US-008..010 — Retry & time bounds:** asymmetric automatic retry, checkpoint-aware opt-in retry for expensive nodes, opt-in time limits.
- **US-011..014 — Reactions:** one-line notifications, approval links, loop terminal reactions, truthful event contract for consumers.
- **US-015..018 — Long-running supervision:** no time-based death, death-resume, silence flags, cooperative control delivery.
- **US-019..022 — Pause & durable wait:** live node repair, rule-driven auto-pause, restart-surviving waits with governed resume, visible waiting inventory with authored escalation.
- **US-023..024 — Target health & quarantine:** per-target breaker, quarantine with repair context and requeue.
- **US-025 — Admission dedupe:** one delivered event, one run.
- **US-026..027 — Manageability & configuration:** agent-operable parity for every state/verb, validated family defaults.

## Core Features

### 1. Failure classification & diagnostics

Every node failure is classified before anything reacts to it: transport/infrastructure (transient), payload-declared (the result body says it failed — semantic), quality/policy rejection (verdict-driven, already owned by gates), authoring error (bad reference, invalid expression — reported with available alternatives), cancellation, and timeout/budget (only where authored). Classification is what the rest of the contract keys on: retry eligibility, breaker accounting, route activation, and repair context all read the class, so a lying "success" or an unclassified error can no longer produce wrong behavior downstream. Tool failures carry remediation hints that travel into repair context and retry feedback. Broken predicates follow a two-default policy — continuation predicates fail open (exit), routing predicates fail closed (routable failure) — and every predicate failure is recorded.

### 2. Failure handling & routing

The precedence chain is the contract's first rule: **node retry → error route → reactions → escalation to generation succession.** Fallible nodes may declare a forward-only error route (mutually exclusive with the success path; a routed failure is "handled") or explicit absorption (allow-fail). With no declaration, failure escalates into the existing generation-repair machinery — the safe default. Silent absorption is impossible by construction. Handled failures never feed target health but always count toward caps and budgets.

### 3. Automatic retry & time bounds

Mechanical nodes (tools, transforms) auto-retry transient-class failures with exponential backoff and a small family-default cap; semantic failures, authoring errors, and cancellations never auto-retry. Expensive nodes (agents, sub-loops) never auto-retry; authored retry on an agent node continues from the latest progress checkpoint when one exists. A failing attempt may carry its own retry-after delay, honored within bounds. Time limits exist only where authored: per-attempt and total-budget bounds with distinct classifications, both routable. Superseded scheduled work (a timer or retry belonging to a state the node already left) never fires — no ghost wakeups.

### 4. Reactions

Authors declare reactions on seven node triggers — `on_success`, `on_error` (fires once, after handling exhausts), `on_retry` (per attempt), `on_pause`, `on_timeout`, `on_cancel`, `on_quarantine` — and, in the loop contract, on each terminal outcome (done, no-op, blocked, failed, exhausted, stalled, canceled — the run-level cancel/kill terminal). A reaction emits a typed event and/or runs a tool, with the node's identity, classified failure context, hints, and resume/approval links pre-bound before input resolution. Effects are fail-open (an effect failure is recorded and never touches the node's outcome), isolated per entry, strictly observe-only (no state mutation), and dispatched only after the state they announce is durably recorded — at-least-once with stable identity for consumer dedupe.

### 5. Long-running supervision

Long work reports progress checkpoints as a side effect of working (its activity stream already exists); the latest checkpoint and its age are visible. Liveness is evidence-based: streamed activity, in-flight tool executions, and transport presence all count as life; duration never triggers anything. Confirmed session death auto-resumes the node from its last checkpoint (continuation with provenance, small attempt bound, then needs-attention). True prolonged silence — no evidence of life across a generous configurable window — raises a needs-attention flag and nothing else. Control verbs (cancel, pause) are delivered to in-flight work through the same liveness exchange; work that never reports is stoppable only by kill.

### 6. Pause & durable wait

Operators pause individual nodes live (in-flight work drains or is cancelled by choice), with provenance, while the rest of the run proceeds; resume offers plain / reset-attempts / immediate variants. Configured auto-pause rules match failure patterns (class, attempts, target) and park nodes instead of letting them burn budget, with rule provenance. Durable waits (timers, events, approvals) survive restarts, carry iteration-scoped identity inside fan-out lanes, admit exactly one resume (single claim, deterministic answers to losers), validate resume payloads against declared expectations, and surface intervention-required when admission keeps failing. Paused/waiting/quarantined states are excluded from stall arithmetic and suspend the run's wall-clock work budget; token spend always counts.

### 7. Target health & quarantine

A breaker per (node family, external target) counts only transport-class failures; an open target fails its nodes fast with an explicit target-unavailable cause that enters the normal failure chain, while other targets keep working; recovery probes close it. A node that exhausts unexpected-failure handling enters quarantine: the run continues on independent branches; the entry carries the full repair context (failure chain, attempts with timestamps, hints, target, input reference); inventories are listable per run/loop/workspace; requeue re-enters through normal succession with all bounds applying. A required-but-quarantined producer surfaces the run as needs-attention naming the dependency.

### 8. Watch-event admission dedupe

Admission of watch-triggered runs derives a stable suppression key from source + event identity + loop, claims it durably in the same act that records the start, and answers duplicates with a structured suppression (recorded diagnostic, durable tombstone). A watch source that cannot provide stable event identity fails validation — never a random fallback, never silent unprotected admission. Overlap policy for *distinct* events is explicitly out of scope (Spec 3).

### 9. Manageability & configuration

Every state and verb above exists with parity on the command line, the programmatic API, and native agent tools: structured output, stable field names, deterministic machine-readable errors naming actual state and allowed transitions, and consistent list surfaces (waiting, quarantine, attention) with pagination and stable ordering. Family defaults — retry caps and backoff, liveness windows, breaker thresholds, auto-pause rules — are configurable at the loop-defaults level, validated on write, resolved specific-over-general (node over loop over defaults), with effective values and their sources inspectable. Running runs keep their admission-time resolution; configuration changes affect new runs.

### Feature interactions

- Classification (1) is the input to retry (3), routing (2), breaker accounting (7), and repair context — it ships first conceptually; everything keys on it.
- The precedence chain (2) sequences retry (3) before routes, routes before reactions (4), and reactions before escalation; cancel (ADR-008) preempts the chain and fires `on_cancel`.
- Death-resume (5) is continuation, not retry (3); the two are distinct in records and bounds.
- Pause (6) suspends retries (3), node clocks (3), stall arithmetic, and the wall-clock budget; quarantine (7) is a terminal-per-node outcome that reactions (4) can announce and succession must respect (needs-attention when required).
- Dedupe (8) runs at admission, before any node contract applies.

## Business Rules

### Precedence & handling

1. Failure handling order is fixed and identical across node families: automatic retry (if eligible) → error route (if declared) → failure reactions → escalation to generation succession. Each step's outcome is recorded.
2. Absorption of a failure requires explicit declaration (error route or allow-fail). Declaring both on one node is a validation error.
3. Error routes are forward-only; a route pointing upstream fails validation (the body remains a strict DAG).
4. A routed or absorbed failure is "handled": it does not feed target health and does not count as unhandled for run policy, but it counts toward iteration caps and budgets.
5. Cancellation preempts handling: no retry fires, no error route activates, `on_cancel` reactions run (cooperative cancel only), and the close reason distinguishes cancel from failure.

### Classification

6. A result body that declares failure fails the node: a non-empty declared error wins over a success indicator; empty error containers fail with an explicit cause; an empty/null error field alone does not fail. A node may declare a result contract naming its own failure shape; built-in rules apply otherwise.
7. Classes and their automatic behavior: transport/infrastructure → retry-eligible; payload-declared and quality rejection → never auto-retried; authoring error → never auto-retried, reported with available alternatives; cancellation → never retried; timeout/budget (authored only) → per rule 14; target-unavailable (open breaker) → fails fast into the normal handling chain, never auto-retried against the open target.
8. Every failure carries an operator-safe cause with an optional remediation hint; hints travel into repair context and retry feedback; all diagnostics are sanitized (secret-redacted, size-bounded, truncation marked) once, before anything downstream reads them.
9. Predicate failure policy: continuation predicates fail open (loop exits, reason names the predicate); routing predicates fail closed (routable failure). Both always recorded. Expressions must compile at validation time; evaluation is cost-bounded, and exceeding the bound is a broken predicate under the same policy.

### Retry & bounds

10. Family defaults: mechanical nodes retry transient failures automatically (exponential backoff, small cap); agent and sub-loop nodes have zero automatic retries. Node-level declarations always win over family defaults; operators tune family defaults.
11. Only transient-class failures are ever auto-retried, on any family, under any configuration default.
12. Every attempt counts toward the node's total budget (if authored) and the run's caps and budgets. No handling path — retry, route, requeue, resume — bypasses run bounds.
13. A failing attempt's self-declared retry-after delay is honored within the node's bounds.
14. Time limits exist only where authored. No node has a default duration limit. Attempt-timeout is transient-class and routable; total-budget exhaustion is a distinct classification. Declared clocks suspend while the node is paused/waiting. Contradictory declarations (attempt limit > total budget) fail validation.
15. Scheduled future work (retry wakeups, authored timeouts, escalation steps) belonging to a superseded node state never fires.

### Liveness & long-running work

16. Health is evidence of life: streamed activity, an in-flight tool execution, or transport presence. Duration alone never triggers any action, flag, or warning.
17. Auto-resume triggers only on confirmed session death (deterministic process/transport loss), never on silence heuristics; ambiguity resolves to the silence path. Resume is continuation from the latest checkpoint (cold start only when none exists), bounded by a small cap, then needs-attention. Resume provenance is recorded and distinct from retry.
18. Prolonged absence of all life evidence across the configured window raises needs-attention and nothing else; the flag clears itself when life returns. A window of zero disables silence evaluation; death-resume is independent of it.
19. Death during pause or wait does not trigger resume (parked work needs no session); the death is recorded and the session restores when work continues.

### Reactions & events

20. Reaction triggers: node-level `on_success`, `on_error` (once, post-exhaustion), `on_retry` (per attempt), `on_pause`, `on_timeout`, `on_cancel`, `on_quarantine`; loop-level one per terminal outcome, firing exactly once per run. Kill fires no node-level reactions; loop-level terminal reactions ride terminal truth and still fire for the outcome a kill produces (see US-013).
21. Effects may emit typed events and/or run tools. Effects are observe-only (no mutation of run state, verdicts, or history), fail-open (failures recorded, never affecting the node), and isolated per entry.
22. Effect context (identifiers, classified failure, hints, resume/approval links) is pre-bound before input resolution and sanitized like all diagnostics.
23. No lifecycle event, reaction emission, or live-stream signal is observable before the state it announces is durably recorded. Delivery after crashes is at-least-once with stable identity; replayed history matches live emissions; observer failures never affect the run.

### Waits, pause, quarantine

24. Waits and approvals never expire by default and are always enumerable with identity, reason, and age. Authored expiry runs its declared escalation ladder in order; expiry without a declared route surfaces needs-attention. A decision arriving mid-escalation wins and cancels remaining steps.
25. Resume admission is single-claim: exactly one resume wins; concurrent attempts receive deterministic queued/already-resolved answers; payloads are validated against the wait's declared expectation; bounded admission failures surface intervention-required. Wait identity is scoped to node + branch + iteration.
26. Pause excludes a node from scheduling and from stall/no-progress arithmetic, carries provenance (actor, reason, manual-vs-rule), and suspends the node's declared clocks. Resume variants: plain, reset-attempts, immediate. Verbs are idempotent with deterministic invalid-state answers.
27. Paused/waiting/awaiting-approval/quarantined time does not consume the run's wall-clock budget; token spend always counts. Stall and no-progress arithmetic count only active work.
28. Auto-pause rules are validated on write, evaluated in deterministic order (first match wins, named in provenance), and never re-fire on the same episode without recurrence.
29. Quarantine is a per-node terminal state that never terminates the run by itself; independent branches continue. Entries carry classified failure chain, attempt history with timestamps, hints, target, and input reference — diagnosable without logs. Requeue re-enters via normal succession under all bounds, with provenance; repeated episodes append to history.
30. Forward progress requiring a quarantined or rule-paused producer surfaces the run as needs-attention naming the dependency — never silent waiting, never unexplained failure.

### Target health

31. Breaker identity is (node family, external target). Only transport-class failures count; semantic and handled failures record as target-health success. Open targets fail bound nodes fast with an explicit cause entering the normal failure chain; recovery probes close the breaker; transitions are recorded and observable.

### Admission

32. Watch-triggered admission derives its suppression key from source + event identity + loop and claims it durably with the run start; duplicates get a structured suppression answer and a recorded diagnostic; tombstones persist across restarts within the source's redelivery horizon.
33. A watch source (built-in or extension-provided) that cannot yield stable event identity fails definition validation naming the gap.

### Manageability & configuration

34. Every state, inventory, and verb in this contract exists on command-line, API, and native agent-tool surfaces with parity, structured output, stable field names, deterministic errors naming actual state and allowed transitions, and paginated stable-ordered lists. Concurrent verb invocations resolve by single claim with the winner's provenance in the loser's answer.
35. Family defaults (retry, liveness windows, auto-pause rules) are configurable at the loop-defaults level, validated on write, resolved node-over-loop-over-default, inspectable with sources, and applied at admission time (running runs keep their resolution). Breaker thresholds and probe cadence are deliberately excluded from that chain: target health is shared state, so its policy is a single daemon-level setting, inspectable with a global source label.

## User Experience

### Personas

Six personas (full definitions in the catalog): loop author, authoring agent, operator, managing agent, approver, extension developer.

### Primary flows

**Author declares failure behavior (US-005..013).** While writing a loop, the author adds an error route to a flaky tool node, `on_error: notify` to a critical one, and a terminal-outcome reaction to the loop contract. Validation catches upstream-pointing routes, unknown tools, and uncompilable expressions immediately with named diagnostics. Nothing else changes — undeclared nodes keep the safe default.

**A transient blip heals itself (US-008).** A tool call fails on transport; the run detail shows the node retrying with attempt count and next-attempt time; the second attempt succeeds; the only trace is the recorded attempt history.

**A crash at 3 a.m. (US-016).** A days-long agent node's session dies. The engine confirms death, resumes from the last checkpoint, and records the continuation. The operator reads the episode in the morning: one death, one resume, minutes lost.

**Live repair without losing the run (US-019, US-024).** A node wedges on a revoked credential. The operator pauses it (the rest of the run continues), fixes the credential, resumes with attempt reset, and the run completes. When a node instead quarantines after repeated unexpected failures, the operator opens the quarantine entry, reads the classified failure chain and hints, repairs, and requeues — watching succession pick it back up.

**Approval where people live (US-012, US-022).** A gate parks the run; `on_pause` posts the approval link to the team channel. The approver clicks, sees the decision context, approves; the run resumes with their identity recorded. Unanswered approvals sit visibly in the waiting inventory with age; an authored ladder re-notifies and escalates.

**Agent-driven supervision (US-026).** A managing agent lists everything waiting, flagged, or quarantined across the workspace, requeues repaired work, cancels a stale run cooperatively, and confirms outcomes — entirely through structured surfaces.

**Author declares the failure contract in the editor (US-028).** While editing a custom loop, the author sets retry/`on_error`/terminal reactions/`wait` in the loop editor. Validation errors block Publish; warnings stay visible and never gate it. Publishing a built-in requires fork first; a rejected publish shows the linter's truth without advancing the version.

**Arrive-and-use hero path (US-029).** An operator opens the loops catalog (roster includes `canceled`), starts a run through the arrive-and-use form (this form is `http`), and reads the loop detail's reliability story and recent runs — all truthful to daemon state, with no invented `stop` control.

### UI/UX considerations

- The run page gains truthful lifecycle surfaces: attempt/retry visibility per node, pause/quarantine/needs-attention states with provenance, waiting inventory with age, target-health indication on failures, and reaction/effect results. All bound to daemon truth (SD-007): no control or metric renders unless the runtime models it.
- Inventories (waiting, quarantine, attention) get first-class list views with filters, age sorting, and empty states; badges surface counts where the shell already surfaces attention items.
- The loop editor gains truthful authoring of the Spec 1 failure contract: reliability envelope (`deadline`/`retry`/`result_contract`/`on_error` with route XOR allow_fail), node triggers and contract terminals as effect lists, `wait` inspector, `on_parent_close`, and a lint dock where errors gate Publish, warnings never do, and zero counts render nothing. Chrome states cover built-in read-only (fork-before-publish) and publish-rejected (422 strip). Bound to daemon/linter truth (SD-007); Start-binding allowlist authoring is held (ADR-018).
- The hero path (catalog → run form → loop detail) is an arrive-and-use composition: catalog roster and filters include the `canceled` terminal; the run form is the `http` start surface with required-input gating and no `stop`; loop detail shows contract, reliability tiles, DAG, and recent runs with truthful status pills.
- Every destructive or state-changing action (kill, requeue, reset-attempts) confirms with the current state it acts on; idempotent answers render as informative, not as errors.
- Accessibility follows the product floor (PRODUCT.md WCAG baseline); state semantics ride the signal palette (information, never decoration).
- Visual Contract authority for these web surfaces is `docs/design/opendesign/loops/` (artboards + `DESIGN-BACKLOG.md` + `DESIGN-LESSONS.md`) per ADR-018.

### Onboarding & discoverability

The failure vocabulary is discoverable from validation: diagnostics name the fields that exist (available-fields listings, suggested node names). Documentation ships with the feature: the loops docs gain a "failure handling" chapter covering the precedence chain, the classification classes, reactions, and the operator verbs, with copy following COPY.md and the glossary.

## High-Level Technical Constraints

- **Sequencing & scope authority.** This spec lands after loops-paper-adoption and inherits its invariants unmodified: prior generations immutable; one coordination authority applies all state transitions; monotonic generation lineage with one live leaf; repair and recovery paths never bypass caps, budgets, revision limits, stall detection, or breaker policy; history namespaces stay read-only; events broadcast only after durable commit.
- **Frozen-surface discipline.** The loop engine files frozen by loops-paper-adoption are extended by new files only.
- **Integration, not invention.** The contract rides existing foundations: the durable run history, the agent session activity stream (liveness/checkpoints), the tool surface (reaction effects), the lifecycle-hook surface (events), channels (notification delivery targets), and the existing validation/linter pipeline (new authoring rules). No second execution engine, no parallel queue, no new event bus.
- **Security & privacy.** All diagnostics, hints, quarantine entries, and reaction payloads pass the single sanitization boundary (secret redaction, size bounds) before any surface reads them. Reaction-invoked tools execute under a defined identity and permission scope (open question 3) and never widen what the loop could already do. No raw ownership tokens on any new surface.
- **Performance from the user's seat.** Liveness evaluation and retry bookkeeping must not perceptibly affect run throughput; inventories must stay responsive at hundreds of entries (pagination is part of the contract).
- **Config lifecycle.** New family defaults enter the existing loop-defaults configuration surface with validation, documentation, examples, and agent-mutable access following the established pattern; the exact key set is the techspec's to define, and every key ships with docs and tests in the same change.

## Non-Goals (Out of Scope)

- **Idea 11 — structured-output re-ask loops:** cut. Generation repair and goal-node turns already own that feedback cycle; a third loop is unpredictable.
- **Idea 12 — graduated stall recovery (fingerprint → bounded reset):** held, not designed. Reopens only if post-repair-context evidence shows generations repeating identically despite repair context. The cheap fingerprint diagnostic may ship earlier as observability, but no reset tier ships in Spec 1.
- **Spec 2 domain:** fan-out completion strategies, windowed width, progress namespace, iteration ergonomics, dead-branch pruning, definition pinning, rollover, payload-ref budgets, cost accounting.
- **Spec 3 domain:** overlap policy for distinct events, catch-up windows, trigger pause-on-failure, ingress verification/projection, the run inbox (signal/update), operator time-travel verbs (rerun-from-node, rewind-to-generation). Admission dedupe (US-025) is the single ingress piece pulled forward.
- **Agent-drivable DAG (idea 35):** future RFC, only after Specs 1–2 land.
- **Node-family state-machine refactor (idea 33):** internal consolidation scheduled after this contract ships; no user-facing surface.
- **Deferred reaction triggers:** `on_resume` and authorable liveness reactions (engine events cover both); state-writing reaction effects (observe-only boundary holds).
- **Start-binding authoring (`start[]` / Start lane / hands-free surface allowlist):** held for Spec 3 adjacency (ADR-018). Ship gate: editor writes `start[]`, production sidebar gains the Start lane, and docs drop read-only strip language — until then Visual Contract material only; Spec 1 keeps current production Start strip behavior.
- **No default expiry, no default time limits, no default agent retries** — these are explicit product decisions (ADR-003/004/005), not gaps.

## Architecture Decision Records

- [ADR-001: Spec 1 scope — the node lifecycle & failure contract slate](adrs/adr-001.md) — 19 ideas in; idea 11 cut; idea 12 held.
- [ADR-002: Escalate-by-default failure posture with explicit absorption](adrs/adr-002.md) — precedence chain; no mandatory annotations; predicate policy; bounds always count.
- [ADR-003: Family-asymmetric automatic retry](adrs/adr-003.md) — mechanical retries transients by default; expensive never blind-retries; classification gates retry.
- [ADR-004: Long-running-first supervision](adrs/adr-004.md) — no default time limits; liveness by evidence; death-triggered bounded resume; false-positive protection.
- [ADR-005: Indefinite visible waits with authorable escalation](adrs/adr-005.md) — no default expiry; visibility as safeguard; waiting suspends work-bounds; governed resume.
- [ADR-006: Authorable reactions](adrs/adr-006.md) — 7 node triggers + loop terminal reactions; emit/run effects; fail-open, observe-only, commit-gated.
- [ADR-007: Target-health isolation and quarantine](adrs/adr-007.md) — per-target transport-only breaker; quarantine with full repair context; needs-attention surfacing.
- [ADR-008: Cancel ≠ kill with parent-close policy](adrs/adr-008.md) — cooperative cancel with reactions; immediate kill; terminate/cancel/abandon for children; typed sub-loop failure boundary.
- [ADR-009: Watch-event admission dedupe](adrs/adr-009.md) — loud durable suppression; stable identity required at validation.
- [ADR-018: Web Visual Contract expansion — editor lifecycle authoring + hero path; start-binding held](adrs/adr-018.md) — Spec 1 absorbs editor + hero path UI; `start[]` authoring stays Spec 3.

## Open Questions

1. **Numeric calibration.** Default retry caps and backoff bounds per family, the liveness silence window, the death-resume attempt cap, breaker thresholds and probe cadence, and predicate cost bounds are qualitative here (small, generous, bounded). The techspec proposes concrete values inside these qualitative bounds, with the calibration rationale recorded.
2. **Approval gating for destructive verbs invoked by agents.** Whether kill and requeue, when invoked by a managing agent (as opposed to an operator), require an approval-policy gate under the existing permission model — to be settled in the techspec against the tool-risk classification.
3. **Reaction-effect execution identity.** Which identity and permission scope a reaction-invoked tool runs under (the run's owner, a dedicated effect scope, or the loop's declared runtime), and how that interacts with workspace boundaries — techspec decision with security review.
