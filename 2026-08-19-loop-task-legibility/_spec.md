# Spec: Loop & Task Legibility

---

# Part I — Product

## Overview

Loops execute through the autonomy kernel by design: every run materializes a coordinator record plus one execution record per action node × fan-out item × round. That design stays. The problem is legibility, in both directions:

- **Task surfaces drown.** Every listing surface (web list/board/dashboard/inbox, command-line listing, programmatic listings, native tools) mixes those mechanical records with human work items as undifferentiated rows full of machine ids. A single active loop adds dozens of near-identical rows; the shipped client-side nesting degrades under pagination back to the flood it was meant to prevent. The owner's verdict: loop rows in Tasks "serve para nada".
- **Loop surfaces underserve.** The run page is an operator cockpit (many competing panels, exception-only node rows, machine ids in primary positions) that fails PRODUCT.md's "approachable first" principle — while still missing the operator's most necessary view: the executing graph. There is no live per-node topology, no complete node roster, no durable full-history timeline, and node→session/record links are partial and live-only. An open P1 bug (re-found 6×) documents operators concluding "stalled" while real progress happened.
- **Lifecycle integrity leaks.** A run that ends abnormally (crash, forced stop) can leave its coordinator and cell records live forever — two such orphans exist in the owner's workspace today. Run-terminal and record-settlement only reconcile on the happy path.

This spec fixes legibility and integrity without touching the execution architecture: classify loop records at the source and remove them from task listings by default; make the Loops area the complete, two-register observability home (plain-language default, operator depth one step deeper, live run graph); and guarantee terminal runs always settle their records, retroactively repairing existing orphans.

**Who it is for:** supervisors (the people whose work the loops do — the default read must not require runtime literacy), operators (the same people diagnosing and intervening), and agents (first-class users operating through structured surfaces).

**Why it is valuable:** it converts loop execution from noise + forensics into calm supervision — which is the product's core promise: agent work legible and controllable at a glance.

## Goals

- After this ships, every task listing surface shows only human-facing work items by default; loop execution records appear only under an explicit filter, visually distinguished, and always linked to their run.
- A supervisor opens any run page and passes the 30-second briefing test: what is running, what needs them, how far along it is, and what it produced — in plain words, no machine ids in the default read.
- An operator sees the run's authored graph rendered live — per-node state, progress, errors, retries, fan-out rollups — plus a complete node roster (healthy nodes included), round history, and a durable full-history timeline.
- Anything awaiting a person (approval, request, quarantine) is unmissable: it leads the page, never collapses, and is actionable in place.
- A run reaching any terminal outcome by any path leaves zero live execution records; pre-existing orphans settle automatically on first boot with an audit trail.
- Agents read the same truth structurally: calm-default listings with explicit include filters, run node/round/timeline reads, and a headless event-follow with resume — with deterministic errors throughout.
- The web id-string parsing contract disappears; loop provenance is structured everywhere it is exposed.

## User Stories

[Full user stories](_user_stories.md) — 20 stories across six areas:

- US-001..004 — Tasks calm default: exclusion by default, opt-in reveal, honest aggregates, structured-listing parity.
- US-005..009 — Run page plain register: briefing test, steps+rounds progress, unmissable needs-you, outcome-first terminal reads, durable narrated story.
- US-010 — Loops at a glance: runs roster in plain words.
- US-011..015 — Operator depth: live run graph, complete node roster, round history, in-place interventions, bidirectional links.
- US-016..018 — Lifecycle integrity: terminal settlement guarantee, retroactive orphan repair, auditable reconciliation.
- US-019..020 — Agent manageability: structured node/round/timeline reads, headless event follow with resume.

## Core Features

### 1. Loop-record classification with calm listing defaults

Loop execution records (coordinator + cells) are classified by provenance at the source and excluded from every task listing surface by default. A hidden-by-default filter (web) and an explicit include parameter (command-line, programmatic, native tools) reveal them — distinguished, in plain words, linked to their run. Aggregates (dashboard, inbox, counts, facets) exclude them. Requires: structured loop provenance on every exposed record (run identity, step, round, item as fields), replacing the web's id-string parsing. Interacts with feature 4: escalations from hidden records surface exclusively through the loop attention lane.

### 2. Loop run read surface: nodes, rounds, timeline, links

A run-scoped read of every authored node × round — state, attempt history, timing, session and record identities, fan-out rollups — powering the graph, roster, history, and progress views from one source of truth. A durable, paged, resumable run timeline replaces the client-window-only story (full history survives reload and long runs). Node ↔ session ↔ execution-record links become bidirectional and persist after the run ends. Agents get the same reads structurally, plus a headless event-follow with resume.

### 3. Terminal settlement guarantee + reconciliation sweep

Every terminal transition of a run — natural, canceled, killed, crash-recovered, child-stopped — settles the run's coordinator and cell records as part of the same transition. A reconciliation sweep (at boot and periodically) converges any divergence within one cycle, is idempotent, repairs pre-existing orphans automatically on first boot, and stamps every repaired record with a structured reconciliation reason distinct from natural completion.

### 4. Two-register presentation

One run page, two registers, progressive disclosure, no mode toggle. Default register: plain-language narrative answering running / needs-you / progress (steps + rounds) / produced — status in meaning, never mechanics; failure and needs-you never collapse. Operator register, one step deeper: live run graph (the spec's largest redesign target), complete node roster, round history, raw events, attempts, ids. The runs roster leads with plain outcomes and needs-you markers. Loop attention keeps its own lane (bell → loop surfaces); Tasks dashboards/inbox only lose loop records (no redesign of those modes).

## Business Rules

- **Classification is by provenance, never by name or id-string matching.** A person's task mentioning "loop" in its title is always a work item; a loop-created record is always an execution record.
- **Exclusion is default-filtering, never erasure.** Loop records remain queryable (explicit filter), linkable (from the run page), and visually distinguished when revealed. No surface may silently drop them from existence.
- **Default parity across surfaces.** Every task listing surface — web, command line, programmatic, native tools — applies the same calm default and honors the same explicit include filter. No surface tells a different story.
- **Aggregates follow the default.** Counts, facets, dashboards, and inbox lanes never include hidden loop records.
- **Escalations never get lost.** A quarantined/attention-flagged cell surfaces through the loop attention lane (attention bell, run page needs-you, node inventory). Exclusion from Tasks must not reduce escalation visibility anywhere.
- **Terminal settlement invariant.** A run in any terminal outcome has zero live execution records; any divergence (crash windows) converges within one reconciliation cycle. Reconciliation is idempotent and audit-stamped with a structured reason distinct from natural completion; active runs are never touched by the sweep.
- **Default register purity.** The default read of loop surfaces contains no machine ids as primary text; status copy reports meaning, never mechanics; failure and needs-you signals never collapse under disclosure.
- **Progress semantics.** Progress = settled action steps out of total action steps in the current round, plus a round counter (hidden on round 1). Fan-out renders as derived rollup counts. Attempts are step metadata, never sibling steps. Route-not-taken and never-materialized branches are excluded from totals and rendered neutrally.
- **Truthful UI (SD-007).** Every rendered state maps to runtime-modeled state; rollups are derived, never entered; no invented controls or metrics.
- **Current-state truth.** Status reads everywhere (run headline, node state, roster) reflect the latest attempt/current execution: a node that recovered on a later attempt never reads by its stale failure, and a transient node failure the run absorbed (retried, rerouted) never paints the run's headline as failed. Attempt history stays fully readable in the operator register — recovery hides nothing, it just wins the headline.
- **Workspace scoping.** Every read and stream in this spec is workspace-scoped; records and events never leak across workspaces under any filter.
- **Verb parity.** Any capability shown in the new views (reads, follows, filters, interventions surfaced) is available through structured agent surfaces with deterministic errors.
- **Interventions validate against current state.** Node verbs offered reflect the node's actionable states; stale or terminal-run interventions are rejected deterministically; concurrent interventions resolve to a single winner.

## User Experience

**Supervisor journey (steady state):** start a run (existing flows) → keep working in Tasks, which shows only their work → the attention bell raises a loop lane when a run needs them → open the run page, read the briefing (status line, needs-you, steps/rounds, story) → act in place (approve/respond) → collect outcome and artifacts when terminal. They never meet a machine id.

**Operator journey (diagnosis):** open the run page → disclose the operator register → read the live graph to locate the hot/failed node → open the node: attempts, error class, timing → jump to its session or execution record → intervene (requeue/amend/cancel) from the roster or graph → watch state converge live. After a crash: boot, and the sweep has already settled terminal leftovers, audit-stamped.

**Agent journey (headless):** list tasks (calm default) → query a run's nodes and rounds → follow events with resume → act through existing verbs → verify terminal settlement by listing records. Deterministic errors at every step.

**Accessibility:** WCAG 2.2 AA floor per PRODUCT.md. State is never carried by color alone (icon/text pairing on graph nodes, roster rows, status pills); complete keyboard paths for disclosure, graph node selection, and needs-you actions; reduced-motion alternative for live graph animation; plain language as an access requirement — runtime jargon only where precision earns it, one step deeper.

**Discoverability:** the default read teaches itself (few elements, obvious next action). The loop-records filter in Tasks is intentionally quiet (hidden by default); the agent path is documented in the official skill and command help.

## High-Level Technical Constraints

- **No second execution engine.** The kernel remains the single execution substrate; this spec adds classification, reads, settlement guarantees, and presentation — never a parallel work queue or executor (institutional directives L-003/L-004/L-005 reaffirmed; owner decision).
- **No new run/node lifecycle states.** Terminal outcomes and live states stay exactly as defined in the glossary; this spec adds no state, only guarantees and projections over existing ones.
- **The kernel stays loop-agnostic.** Classification must not teach the task kernel loop-specific vocabulary (existing kernel-neutrality decision); provenance rides the neutral discriminators that already exist.
- **Existing surfaces evolve, not fork.** The run page, runs roster, node inventory, and Tasks app are redesigned/adjusted in place — no parallel "simple" routes or duplicate views.
- **Agent/operator manageability outcome:** everything the new views read — classified listings, node/round rosters, timelines, follows, settlement/audit evidence — must be inspectable and operable outside the web app through structured command-line and programmatic surfaces with deterministic errors; repair (sweep) must be observable and its evidence queryable.
- **Extension ecosystem expectation:** consumers of task listings (bridges, native tools, extensions reading runtime surfaces) inherit the calm default and the explicit include filter with no bespoke carve-outs; no new extension points are expected, and the final design must state extension impact explicitly.
- **Performance from the user's seat:** the run page briefing renders fast on long runs (history loads on demand); the live graph stays responsive on wide fan-outs via rollups; listings stay paginated with stable ordering.
- **Privacy/security posture unchanged:** existing redaction rules apply to every new read and stream; no secret material appears in timelines, rosters, or audit reasons.

## Non-Goals (Out of Scope)

- **A separate loop execution engine or queue** — rejected permanently (kernel remains the substrate).
- **Redesign of the Tasks dashboard and inbox modes** — they only stop counting loop records; their own redesign is a separate effort.
- **Loop DSL, editor, and authoring-surface changes** — the builder canvas and definition flows are untouched (the run graph is a read-only observability surface; whether rendering internals are shared is a technical choice, not a surface change).
- **New retry/attempt semantics** — settlement fixes lifecycle integrity; how loops retry nodes is unchanged.
- **Cross-run analytics** (per-node failure rates across runs, trend dashboards) — identified as valuable, deferred to its own effort.
- **Notification/digest system changes** — the attention bell keeps its existing contract; this spec only guarantees loop escalations continue flowing through it when records leave Tasks.
- **Session page changes** — links land on the existing session surfaces.

## Open Questions

None — every Stage 1 fork was resolved in the grill (ADR-001..003 record the significant ones: exclusion-by-default, two-register single page, live run graph in scope). Surface-level naming (filter name, verb spelling) belongs to Stage 2's DX contract.

---

# Part II — Technical

## Executive Summary

Forensic frame — confirmed 2026-08-19 on the owner's dev daemon: two runs of `revisao-paralela` crashed at ~17:03:58 and ~17:04:09; `compozy task list` still showed both "Loop coordinator revisao-paralela" tasks QUEUED 15 minutes later with cell subtasks pending — run terminal, records live, permanently (no reverse sweep exists; reconciliation is happy-path-only at `internal/store/globaldb/global_db_task_coordinator_settlement.go:16`). The same listing interleaved those records with the operator's real work items. Separately, `docs/qa/bugs/BUG-20260719-autonomous-progress-unobservable.md` (P1, re-found 6×) documents operators unable to distinguish autonomous progress from a stall.

The design adds **no execution machinery**: classification and filtering ride existing provenance columns (ADR-004); run observability becomes three computed projections — on-demand node roster, pure-function briefing, durable paged timeline — over tables that already exist (ADR-005); lifecycle integrity becomes an inline settlement rule plus an idempotent sweep (ADR-006); and the web re-renders both surfaces under the two-register contract (ADR-002/003). Primary trade-off taken: computed reads over materialized projections — one source of truth, no second state to reconcile, bounded by run-scoped pagination.

## MVP Boundary

MVP = all four fronts plus the QA tail: (1) catalog classification + typed include filter with 4-surface parity and web Tasks cleanup; (2) run read layer — roster, briefing, durable timeline — with `loop why/nodes/events` CLI verbs; (3) terminal-settlement invariant + reconciliation sweep + retroactive repair; (4) run-page two-register redesign incl. live DAG, node roster, generation history, runs-roster re-rank. Post-MVP (explicitly deferred): cross-run analytics, dashboard/inbox redesign, run-health-as-indexed-attribute rollups (Temporal A5), timeline `GROUP BY` aggregations. Out of scope permanently: separate loop executor, DSL/editor changes, retry-semantics changes.

## Developer Experience

- [Developer experience contract](_dx.md) — CLI (`task list --include-loop/--loop-run`, `loop nodes --run --all`, `loop why`, `loop events --follow`), HTTP/UDS routes (`/nodes`, `/briefing`, `/timeline`, tasks query fields), `config.toml` (`[loops] reconcile_interval`), native tool (`compozy__task_list`), deterministic errors.
- [UI change map](_uiux.md) — 7 surfaces with default-read contracts, states, component plan, six artboards under `docs/design/opendesign/loop-legibility/` (`loop-legibility-tasks-list.html`, `loop-legibility-run-default.html`, `loop-legibility-needs-you.html`, `loop-legibility-run-dag.html`, `loop-legibility-run-roster.html`, `loop-legibility-runs-roster.html`) plus `DESIGN-NOTES.md` / `loop-legibility.css` / `index.html`.

## System Architecture

| Component | Purpose | Boundary |
|---|---|---|
| Catalog classification | Default exclusion + typed include + provenance projection in the task catalog read model | `internal/task` (neutral query fields) + `internal/api/core` (mapping owns the `loop-coordinator` constant) + `internal/store/globaldb` (SQL predicate/join/projection) |
| Run read layer | Roster (node×generation, on demand), briefing (pure verdict), timeline (paged over `loop_run_events`, notable/all views) | `internal/loop` (assembly + verdict + view map) + `internal/api/contract` payloads + `internal/api/{httpapi,udsapi}` routes + `internal/cli` verbs |
| Settlement + sweep | Inline settlement on every terminal path; `RunReconciler` at boot + interval; audit reasons on `task_events` | `internal/store/globaldb` (settlement helpers) + `internal/loop` (sweep queries/logic) + `internal/daemon` (wiring, ticker) |
| Web two-register | Tasks cleanup (delete regex/nesting; reveal filter) + run-page default read + operator register (DAG/roster/history) + runs roster re-rank | `web/src/systems/tasks`, `web/src/systems/loops` |

Data flow: web and agents read the same three projections; the existing per-run SSE stream accelerates (invalidate/append) and every rendered state reconciles against paged reads. No new streams; no new queues.

## Architectural Boundaries

- **No new Go packages.** All backend work lands in existing packages; `internal/daemon` remains the only composition root (sweep ticker wired there). `mage Boundaries` unchanged.
- `internal/task` stays loop-noun-free: it gains only the neutral `CatalogQuery.ExcludeCreatedBy []ActorRef` plus `CatalogQuery.LoopRunID` (the one acknowledged correlation name, mirroring `task_runs.loop_run_id`). The string `"loop-coordinator"` lives in `internal/api/core` (mapping) and `internal/daemon` (existing seed constant) — never in `internal/task`.
- `internal/loop` may not import `internal/api/*` or `internal/daemon` (unchanged); the briefing/roster/timeline types live in `internal/loop`, wire shapes in `internal/api/contract`.
- Web: cross-system imports only via `@/systems/<domain>`; the DAG renderer lives in `web/src/systems/loops/`, reusing only pure layout/geometry utilities from the editor (no authoring chrome, no editor imports into run views beyond shared pure libs).

## Implementation Design

### Core Interfaces

```go
// internal/task — neutral catalog additions (kernel stays loop-noun-free)
type CatalogQuery struct {
    // …existing fields unchanged…
    ExcludeCreatedBy []ActorRef // rows whose created_by matches any ref are excluded (SQL predicate)
    LoopRunID        string     // scopes via task_runs.loop_run_id join; implies inclusion of matches
}
```

```go
// internal/loop — run read layer (computed projections; no writes)
type RunReadService interface {
    NodeRoster(ctx context.Context, ws string, runID RunID, q RosterQuery) (RosterPage, error)
    Briefing(ctx context.Context, ws string, runID RunID) (Briefing, error)
    Timeline(ctx context.Context, ws string, runID RunID, q TimelineQuery) (TimelinePage, error)
}

type RosterQuery struct{ State NodeStateFilter; Generation int; Cursor string; Limit int }

type Briefing struct {
    RunID    RunID
    Status   RunStatus
    Tone     BriefingTone // ok | needs_you | degraded | failed
    Headline string       // projection of the typed fields below — never the only carrier
    Detail   string
    Blockers []Blocker    // kind, ids, waiting_since, unblocker (exact CLI command)
    Outcome  *RunOutcome  // terminal runs only: status, cause, actor_kind/ref (cancel/kill), at
    Artifacts []RunArtifact // name, output id, ref, availability: available|partial|pruned
    Progress StepProgress // round, steps_done, steps_total
    Usage    RunUsage     // tokens, cost_usd, budget_used_pct, duration
}
```

```go
// internal/store/globaldb — THE single cause-aware terminal-settlement primitive (B-003).
// Every transition that sets a loop run terminal — plan-driven, cancel, kill, child-stop,
// crash-recovery classification, sweep repair — calls this inside the same transaction.
// No other code path mutates loop execution records to a terminal state.
func settleLoopRunTerminal(ctx context.Context, tx taskpkg.Tx, runID string, cause loop.TerminalCause) (loop.SettleResult, error)

// internal/loop
type TerminalCause string // done | noop | failed | exhausted | stalled | canceled | killed | run_missing
type SettleResult struct{ CellsSettled, RunsCanceled int; CoordinatorStatus taskpkg.TaskStatus }
```

```go
// internal/loop — reconciliation (observes + settles via settleLoopRunTerminal; claims nothing)
type RunReconciler interface {
    // NeutralizeOrphans runs synchronously at boot BEFORE any claimer starts:
    // one transaction removing work-eligibility of every execution record owned
    // by a terminal or missing loop run (B-004 barrier).
    NeutralizeOrphans(ctx context.Context) (SweepReport, error)
    SweepOnce(ctx context.Context) (SweepReport, error) // idempotent; interval safety net
    BackfillProvenance(ctx context.Context) (int, error) // metadata-only; covers ACTIVE runs too (B-005)
}
type SweepReport struct{ RunsExamined, RecordsSettled, OrphansRepaired int }
```

### Data Models

**No schema changes** to `tasks`/`task_runs`/loop tables (ADR-004/005/006). Changes are data, payload, and config:

| Field | Shape | Purpose |
|---|---|---|
| coordinator task `metadata_json` (+`loop_run_id`,`loop_name`,`workspace_id`) | existing JSON column, new keys at seed (`global_db_loop_coordinator_seed.go`) | closes the coordinator provenance gap; historical rows repaired by the **provenance backfill pass** (B-005): metadata-only, idempotent, relational (joins `task_runs.loop_run_id` → `loop_runs`), runs at boot and covers ACTIVE runs too — provenance repair is not lifecycle settlement. Rows whose run row is gone AND unreconstructible render the truthful degraded shape: `loop{run_id (from the run's loop_run_id), role (from run_kind)}` with `loop_name` absent — never id/title parsing |
| `LoopProvenance` (one shared optional wire type) | `{run_id, loop_name?, role: coordinator\|cell, generation?, node_id?, item_index?}` — `loop_name` omitted (Go omitempty / TS optional) when the owning run was retention-deleted and unrecoverable | projected on **both** `TaskCatalogItemPayload.loop` and the single-task `TaskPayload.loop` (B-003) — deep links never depend on list navigation; replaces the web id-regex everywhere |
| `LoopRunListItem` additions (`GET /workspaces/:ws/loop-runs`) | `attention?{kind, count, since}` + `progress{round, steps_done, steps_total}`; server ordering needs_you → active → terminal applied **before pagination** | the runs roster (S6) is a server-owned read — no client-side page sorting, no N+1 briefings (B-001); MVP operational summary, NOT the deferred cross-run analytics |
| `Briefing.Outcome` + `Briefing.Artifacts` | typed terminal outcome (`status, cause, actor_kind?, actor_ref?, at`) + artifact collection (`name, output, ref, availability: available\|partial\|pruned`) | terminal briefings carry results as machine truth (B-002); sources: loop outputs + output blobs; existing redaction applies; CLI text ("Produced: …") is a projection of these fields |
| `GET …/loop-runs/:id/nodes` → `LoopRunNodesResponse` | `{run_id, loop_name, nodes[], fanout_rollups[], next_cursor}`; node = `{generation, node_id, item_index, state, attempt, attempts[], next_retry_at?, child_loop_run_id?, cancellation?{disposition, actor_kind, actor_ref, cause}, started_at, ended_at, session_id, cell_task_id, usage?}` | roster read; `attempts[]` exposes `loop_node_attempts` (first public exposure — per-attempt `state/failure_class/disposition/ended_at`); `next_retry_at`/`child_loop_run_id`/`cancellation` close the operator-data gaps (B-002) |
| `GET …/loop-runs/:id/briefing` → `LoopBriefingResponse` | mirror of `Briefing` above | verdict parity for page + `loop why` |
| `GET …/loop-runs/:id/timeline` → `LoopTimelineResponse` | `{entries[{seq, kind, generation?, node_id?, attempt?, title, at}], next_cursor}` | durable paged story; `title` server-rendered in meaning register |
| `config.toml` `[loops] reconcile_interval` | duration, default `"1m"`, must be positive | sweep cadence; boot sweep unconditional; joins the existing `[loops]` section |

**Roster projection contract (B-002 — closed enum + total mapping):** the roster's `state` is a closed projection enum = the 14 persisted output states (`pending, enqueued→queued, running, retrying, waiting, paused, awaiting_child, control_pending, awaiting_goal, succeeded, partial, failed, canceled, quarantined`) **plus derived `not_taken`**. Precedence per node×generation: node controls (canceled/canceling/paused) and waits overlay the output state; a scheduled retry projects `retrying` with `next_retry_at`. **`not_taken` derives only from durable route evidence** — a `route_taken` event selecting a different arm, a `branch_pruned` event, or a recorded gate/route cause — never from the mere absence of an output row. An authored, reachable node with no output row and no route evidence projects **`pending`** (the `_dx.md` example's downstream `saida`). Every persisted state maps; the mapping is total and unit-tested exhaustively.

**Terminal settlement cause→status matrix (B-003):**

| Terminal cause | Coordinator task | Live (non-terminal) cells + runs | Already-terminal cells |
|---|---|---|---|
| `done` / `no-op` | completed | canceled — reason detail "run done; node no longer needed" | untouched (truthful outcomes preserved) |
| `failed` / `exhausted` / `stalled` | failed | canceled — reason detail carries the run outcome (the cell itself did not fail) | untouched |
| `canceled` / killed | canceled | canceled | untouched |
| run row missing (retention) | canceled — reason `run_missing` | canceled — reason `run_missing` | untouched |

Children settle before the coordinator inside the one transaction; queued/claimed runs of settled records are canceled in the same transaction.

**Side-table-vs-JSON decisions:** classification predicate = existing **columns** (`created_by_*` — matchable state stays columnar; L-003); run scoping = a correlated **`EXISTS` semi-join** on `task_runs.loop_run_id` (one row per task across multi-attempt runs, applied before ordering/pagination — B-006); display provenance = existing **JSON** (`metadata_json` — opaque display data, never SQL-matched; the one SQL touch is the recorded ADR-004 fallback); timeline = existing **table** (`loop_run_events`); notable/activity/chatter view = **authored Go map** over event kinds (code, not storage — it is presentation classification, not state; exhaustive over the event-kind CHECK list, enforced by test); roster = **computed**, no storage.

### API Endpoints

- `GET /api/tasks` (+ UDS + `compozy__task_list`): new typed query fields `include_loop` (bool, default false), `loop_run_id` (implies include). Handler maps to `CatalogQuery.ExcludeCreatedBy`/`LoopRunID`; facets/counts computed over the same filtered set; `parent_task_id` unchanged (returns children regardless — explicit parentage). Errors: `400 invalid_query_field`.
- `GET /api/workspaces/:ws/loop-runs/:id/nodes`: filters `state|generation|cursor|limit`; `not_taken` derived from the pinned definition snapshot; `404 loop_run_not_found`, `400 invalid_node_state|invalid_cursor`.
- `GET /api/workspaces/:ws/loop-runs/:id/briefing`: no params; `404` only. Terminal briefings populate `outcome` + `artifacts` (typed; redacted per existing rules).
- `GET /api/workspaces/:ws/loop-runs` (existing list, extended — B-001): items gain `attention?` + `progress`; ordering needs_you → active → terminal is applied server-side **before pagination** (stable across pages); same fields on HTTP, UDS, and `compozy loop runs` structured output; existing filters unchanged. Multi-page ordering covered by integration tests.
- `GET /api/tasks/:id` (existing single-task read, extended — B-003): carries the shared optional `LoopProvenance` as `loop`, identical to the catalog item's.
- `GET /api/workspaces/:ws/loop-runs/:id/timeline`: `view=notable|all`, `cursor`, `limit`. **Snapshot-fenced backward pagination (B-004):** the **first page is the newest window** in display order and always returns `head_seq` — the run's highest sequence at that read (`0` for an event-less run). Older history pages backward on demand; the opaque `cursor` binds `{run_id, view, fixed_head_seq, before_seq}`, so the page set is immutable under concurrent appends (no moving-head chase, no full-history download on long runs). Cursor replay against a different run (incl. forks/reruns, which mint new run ids) returns `409 timeline_branch_changed`. **Two position types, deliberately distinct:** that opaque HTTP cursor vs. the CLI `--after <seq>` — a plain per-run sequence scoped by the run named in the command (beyond-head → deterministic error; no cross-run token at the CLI). `view=all` is the ordered union of the three internal tiers; heartbeat-class runs of events coalesce **server-side at read**: a coalesced entry spans `first..last` raw sequences and carries `seq = last`, so resume never replays folded events.
- **Durable→live handoff (the seam that makes no-gap/no-duplicate true):** a live reader (web story, `--follow`) takes **one** first page, then attaches the existing SSE stream with `after_sequence=head_seq` (the stream's native resume; valid for `head_seq=0`). New events arrive by push while older history backfills on demand; readers de-dupe by `seq` (strictly monotonic per run). A terminal `status_changed` event closes follows cleanly. Race coverage: append-between-first-page-and-subscribe, continuous appends during backward paging, empty-run seam, long-history lazy backfill.
- CLI: `compozy loop why|nodes --run --all|events --after --follow --view` and `task list --include-loop --loop-run` per `_dx.md` (exit codes and messages there are the contract). All routes registered on HTTP and UDS in the same change (no partial surface).
- Existing routes unchanged: run detail, node verbs, requests, SSE events stream (stays the push channel).

## Integration Points

None outside the daemon — no external services. (Sessions/artifacts are internal links.)

## Impact Analysis

| Component | Impact | Description / risk | Action |
|---|---|---|---|
| `internal/store/globaldb` catalog SQL | modified | exclusion predicate + run join + metadata projection; low risk (indexed) | extend `taskCatalogBaseFilter/Columns` + queries |
| `internal/store/globaldb` settlement | modified | ONE cause-aware primitive (`settleLoopRunTerminal`) called by every terminal path; medium risk (transactional) | new primitive wraps/replaces direct `settleCompletedTaskHierarchyWithExecutor` use on loop paths; child-first ordering; transition-path coverage test fails on any bypass |
| `internal/loop` | new files | roster/briefing/timeline/view-map/sweep; medium | new single-responsibility files (500-line cap) |
| `internal/api/{contract,core,httpapi,udsapi}`, `internal/cli` | modified/new | 3 new routes + fields + 3 CLI verbs; codegen co-ship | `make codegen`; E2E mocks per L-007 |
| `internal/daemon` | modified | sweep wiring (boot + ticker); config plumb | composition root only |
| `web/src/systems/tasks` | modified | exclusion default + reveal filter + provenance; **delete targets below** | |
| `web/src/systems/loops` | modified/new | run-page two-register + DAG/roster/history + roster re-rank | design pass (artboards) precedes |
| `docs/qa` | modified | new/reset scenarios; `TA-web-task-list-loop-subtask-nesting` superseded | QA tail |

**Delete targets (no fallback / no compat shim / no placeholder):**

- `web/src/systems/tasks/lib/task-formatters.ts` — `LOOP_CELL_TASK_ID`/`LOOP_COORDINATOR_TASK_ID` regexes, `parseLoopTaskId`, loop branches of `taskShortId`.
- `web/src/systems/tasks/lib/task-hierarchy.ts` — loop nesting in `buildTaskListTree` + `subtaskSummaryLabel`; `components/task-subtask-list.tsx` loop path (component deleted if loop-only).
- Run-page reliance on the SSE frame buffer as sole story history (`MAX_STORY_FRAMES` remains a render cap only); cockpit sections demoted per `_uiux.md` S4/S5.
- `docs/qa/scenarios/TA-web-task-list-loop-subtask-nesting.md` — superseded by the exclusion contract (retired, replaced by new scenarios).
- No dual default anywhere: `include_loop` absent = excluded, on every surface, same release.

## Compozy Impact Audit

- **Native tools**: `compozy__task_list` gains typed args `include_loop`/`loop_run_id` → descriptor + input-schema digest + tests updated in the same change; availability diagnostics and capability gates unchanged (read-only listing tool). Checked surfaces: the full `internal/daemon/native_*` inventory — no other native tool reads task listings or loop runs today (no `compozy__loop_*` toolset exists); no risk-flag changes.
- **Extensibility and hooks**: no new hooks; settlement/audit events are emitted at the owning call sites (`settleLoopRunTerminal`, sweep) — never by tailing event tables. Extension Host API method set unaffected (checked the closed method list — no task-listing or loop-run read methods exist); `loop.watch_source` provide surface unaffected; bridge SDK unaffected (bridge task subscriptions target terminal notifications by task id, not listings — checked `bridge_task_subscriptions` contract); MCP sidecars unaffected. Config lifecycle: one added key (`[loops] reconcile_interval`, see Config Lifecycle). Registries: none touched.
- **Workspace data isolation**: every new datum classified — catalog provenance (`loop` object) is **workspace-scoped** (rides `tasks` rows; CLI resolves workspace context via existing task parsing, HTTP/UDS take the existing `workspace` param, native binder takes the existing workspace arg, store predicates keep `workspace_id`); roster/briefing/timeline entries are **run-scoped within a workspace** (routes live under `/workspaces/:ws/loop-runs/:id/*`; store queries join `loop_runs.workspace_id`; existing SSE stream stays workspace-guarded); cursors carry `{run_id, seq}` only — a foreign-workspace run id 404s before any cursor evaluates; settlement/audit events are **task-scoped** rows carrying `workspace_id` correlation keys. Web query keys include the workspace per existing convention — the new query-option factories must too (asserted in web tests). Proof paths: IT-015 (4-surface parity), IT-016 (cross-workspace denial on every new read), catalog SQL predicate tests.
- **Official Compozy skill**: `skills/compozy/` UPDATE REQUIRED — the task-listing reference gains the calm default + `--include-loop`/`--loop-run` contract; the loops reference gains `loop why`, `loop nodes --run --all`, `loop events --after/--follow/--view`, and the settlement audit reasons (`loop_run_terminal` / `reconciled_run_terminal` / `run_missing`). Co-ships in the same change as the CLI verbs.

## Extensibility Integration Plan

- **Native tools**: `compozy__task_list` gains `include_loop`/`loop_run_id` (schema digest + descriptor update + tests). No other `compozy__*` tool touches task listings (checked `internal/daemon/native_*`).
- **Bridges**: bridge task subscriptions target terminal notifications by task id (`bridge_task_subscriptions`) — unaffected by listing defaults (checked `internal/bridges`/`internal/notifications` contract). Bridge SDK unchanged.
- **Extensions/hooks/MCP**: no extension surface reads task listings today (checked `internal/extension` host API method set); `loop.watch_source` provide unaffected; no new hooks (settlement emits existing task events; sweep logs are observability, not hooks). No new extension points.
- **Official Compozy skill (`skills/compozy/`)**: UPDATE REQUIRED — document calm default + `--include-loop`, `loop why/nodes/events`, settlement audit reasons.
- **Protocol docs / OpenAPI**: regenerated (`make codegen`); generated TS types + site CLI/API references co-ship.

## Agent Manageability Plan

Everything the views render is agent-operable (consistent with `_dx.md`): `compozy task list` calm default + typed include (`-o json`); `compozy loop why` (verdict + unblocker commands); `compozy loop nodes --run --all` (roster incl. attempts); `compozy loop events --after --follow --view` (durable resume; `-o jsonl`); settlement audit via `compozy task timeline` reasons (`loop_run_terminal` vs `reconciled_run_terminal` vs `run_missing`); deterministic errors per `_dx.md` Errors table. Status discovery: briefing tones and node states are closed enums documented in generated references.

## Config Lifecycle

- **Added**: `[loops] reconcile_interval` (duration, default `"1m"`) on the existing `[loops]` section (`internal/config/loops.go`): struct field + default + validation (positive; `0s` rejected with `"reconcile_interval must be positive"`) + merge/overlay + example in config docs + generated CLI/site reference + validation tests. Boot sweep runs regardless of value.
- **Checked, unaffected**: no existing `config.toml` keys change or retire; no settings UI exposure required (operator-tunable via file; not a UI setting — checked `internal/settings` projection list).

## Testing Approach

Strategy (cases live in `_tests.md`): unit = pure functions and predicates (verdict cascade table-driven; notable/all map; roster assembly incl. `not_taken`; catalog SQL filter/join; sweep idempotency guards). Integration = every terminal path settles in-transaction (natural/cancel/kill/child-stop/crash-classified); sweep boot repair incl. provenance backfill + `run_missing`; roster/timeline pagination stability under concurrent appends; 4-surface parity — **semantic, not byte-equivalent**: identical filtered item sets, provenance values, defaults, cursor behavior, and deterministic error meaning across CLI/HTTP/UDS/native tool, while transport envelopes may differ (N-003). E2E runtime = crash → boot → zero live records; `loop events --follow` resume without gaps. E2E web (Playwright) = tasks default/reveal; run-page briefing/needs-you/terminal states; DAG states incl. reduced-motion. Contract co-ship: OpenAPI + generated TS + `acpmock`/E2E matchers in the same change (L-007). Fakes at I/O boundaries only; real SQLite in integration (fresh + reopen).

## Development Sequencing

### Build Order

1. **Settlement invariant + sweep** (correctness first; unblocks clean QA everywhere) — the cause-aware primitive, inline settlement on every terminal path, and the **boot barrier**: `NeutralizeOrphans` runs synchronously in daemon boot BEFORE `loopActionRuntime.Recover`, the coordinator backstop, the reconciler ticker, or any claim traffic starts (L-003: boot recovery precedes wake/claim). The barrier **fails closed**: on error, readiness is not reported and none of those components start — structured startup error, no log-and-continue (that policy belongs to interval sweeps only). Heavier repair (audit backlog, `BackfillProvenance`) continues after readiness without blocking it. Gate: integration suite over all terminal paths + boot repair + the sweep-vs-`ClaimNextRun` race test + the barrier failure-path test.
2. **Catalog classification + 4-surface parity + web Tasks cleanup** (incl. delete targets) — gate: parity tests + Playwright tasks scenarios + `make codegen-check`.
3. **Run read layer** (roster/briefing/timeline + CLI verbs) — gate: read-layer integration + CLI E2E.
4. **Web run-page two-register** (default read, then operator register: DAG/roster/history; build to the landed set at `docs/design/opendesign/loop-legibility/`) — gate: Playwright run-page scenarios + visual contracts.
5. **QA pair** (`qa-report` + `qa-execution`) — gate: `make gate-full` + scenario walks.

Phases 1–3 are backend-independent of 4; 2 and 3 may interleave after 1.

### Technical Dependencies

None external. Internal: phase 4 consumes phase 3's reads; the visual contract set at `docs/design/opendesign/loop-legibility/` is already landed and binding for tasks 04/05.

## Monitoring and Observability

- Sweep: structured `slog` per cycle (`runs_examined, records_settled, orphans_repaired, provenance_backfilled, duration_ms`) + per-repair task event (correlation keys: `task_id, run_id, loop_run_id, workspace_id, actor_kind=daemon, release_reason`).
- Reconciler lifecycle (N-003): one daemon-owned goroutine — started by the composition root after the boot barrier, context-aware store calls, stopped and joined on daemon shutdown; cycles never overlap (a tick is skipped while the previous cycle runs); a failed cycle emits a structured error event and retries on the next tick (no tight-loop backoff needed at a 1m cadence).
- Settlement events on `task_events` with `reason` (`loop_run_terminal` | `reconciled_run_terminal` | `run_missing`) — the coverage-matrix test asserts every terminal path emits.
- Read layer: no new metrics (request logs suffice); timeline/roster handlers log only on error.

## Technical Considerations

### Key Decisions

- **DAG renderer**: read-only renderer in `systems/loops`, sharing only pure layout/geometry utilities with the editor canvas — no authoring chrome (T3; ADR-003/005).
- **Story hard cut**: durable timeline is the story's source; SSE accelerates; frame-buffer-as-history deleted (T4; ADR-005).
- **Kernel-neutral mapping**: `include_loop → ExcludeCreatedBy{daemon/loop-coordinator}` lives in `internal/api/core`; `internal/task` sees neutral fields only (ADR-004).
- **Provenance repair is not lifecycle settlement** (B-005): `BackfillProvenance` is a separate, metadata-only, idempotent relational pass (joins `task_runs.loop_run_id` → `loop_runs`) that also covers coordinators of ACTIVE runs — the sweep's never-touch-non-terminal rule binds lifecycle mutation only. One pass, zero legacy (ADR-006).
- **Cursor identity**: timeline cursors embed `{run_id, seq}`; fork/rerun mint new run ids, so branch checks reduce to run-identity equality (Temporal's branch-token lesson, simplified by our id model).
- **Usage fields are truthful-or-absent**: roster/briefing `usage` renders only what the runtime tracks today (run-level tokens/cost/budget; per-node when session accounting provides it) — absent, never fabricated (SD-007).

### Known Risks

- Roster assembly cost on wide fan-out → pagination + rollups; indexes exist; ADR-004/005 record the materialization/generated-column fallbacks.
- Verdict cascade drift vs UI expectations → single served verdict (no client re-derivation) + table-driven tests.
- DAG layout on deep/wide graphs → design pass owns layout grammar; roster remains the always-works fallback view.
- Settlement added to hot terminal paths → reuses existing helpers in the same tx; integration suite guards regressions.

## Safety Invariants

1. **One settlement authority.** Every transition that sets a loop run terminal calls `settleLoopRunTerminal` inside the same store transaction — children first, coordinator last, statuses per the cause→status matrix (killed work is never recorded "completed"; already-terminal children keep their truthful outcomes). No other code path mutates loop execution records to a terminal state; a transition-path coverage test fails when any terminal mutation bypasses the primitive.
2. **Boot barrier before claim traffic — fail closed (B-005).** `NeutralizeOrphans` completes synchronously at boot before `loopActionRuntime.Recover`, the coordinator backstop, the reconciler ticker, or any claimer starts; `task.Service.ClaimNextRun` never returns a run owned by a terminal or missing loop run. If neutralization errors, the daemon **does not report readiness and starts none of those components** — it surfaces a structured startup error and exits/retries boot; the periodic log-and-retry-next-tick policy applies to interval sweeps only, never to the boot barrier. Proven by real-SQLite race and failure-path tests.
3. The sweep is idempotent: a re-run settles nothing twice and emits no duplicate audit events (status-guarded writes).
4. The sweep never performs lifecycle mutation on records of non-terminal runs. The provenance backfill is exempt precisely because it is metadata-only, idempotent, and never touches status, runs, or lifecycle fields.
5. Sweep and live settlement serialize through the store transaction (`BEGIN IMMEDIATE`); exactly one winner per record.
6. Reconciliation reasons are structured and distinct: `loop_run_terminal` (inline) ≠ `reconciled_run_terminal` (sweep) ≠ `run_missing` (retention-orphan) — queryable, not prose.
7. **Timeline positions.** The first page is a newest window carrying `head_seq` (0 for an empty run); older pages are snapshot-fenced — the opaque cursor binds `{run_id, view, fixed_head_seq, before_seq}`, so concurrent appends never move a page set. Cursor/run mismatch returns `409 timeline_branch_changed`, never spliced histories. CLI `--after` is a per-run sequence by construction with a deterministic beyond-head error. The durable→live seam is one first page + SSE `after_sequence=head_seq`, de-duped by monotonic `seq` — no full-history read is ever required to go live.
8. Catalog exclusion is a server-side SQL predicate; facets, counts, and every listing surface (CLI/HTTP/UDS/native tool) compute over the same filtered set — no client-side filtering, no divergent defaults.
9. Classification matches provenance columns only (`created_by`, `task_runs.loop_run_id`) — never id or title strings, at write time and at repair time alike.
10. Every new read/stream is workspace-scoped at the query layer; cross-workspace ids resolve to `404`, never empty success.
11. The sweep observes and settles via the settlement primitive; it never claims runs — `task.Service.ClaimNextRun` remains exclusive (L-005).
12. The briefing verdict is computed server-side from the same reads the page renders; web never re-derives a different verdict.
13. Existing redaction rules apply to every new payload — no `claim_token`, secrets, or session credentials in roster, briefing, timeline, or audit reasons.
14. **Roster truth.** `not_taken` projects only from durable route evidence (`route_taken` electing another arm, `branch_pruned`, recorded route causes); a reachable node with no output row projects `pending`. The two are never conflated, and the source→projection mapping is total (exhaustively tested).

## File References

### Repo Files

- `internal/store/globaldb/global_db_task_catalog_sql.go:308-346` — catalog columns + the single implicit filter; where exclusion/join/projection land.
- `internal/task/catalog.go:57-73` — `CatalogQuery` to extend (neutral fields).
- `internal/api/core/task_catalog.go:16-46` — query parsing; where `include_loop` maps to neutral fields.
- `internal/store/globaldb/global_db_loop_coordinator_seed.go:90-181` — coordinator seed (metadata gap) + id grammar.
- `internal/loop/coordinator_metadata.go:20-50` + `internal/loop/constants.go:4-7` — cell metadata keys to project.
- `internal/store/globaldb/global_db_task_coordinator_settlement.go:9-42` + `global_db_task_parent_rollup.go:20-160` — existing settlement/rollup helpers the invariant reuses.
- `internal/store/globaldb/global_db_task_coordinator_mutations.go:18-51` + `global_db_loop_cancel_finalize.go:13-70` — terminal paths that must gain inline settlement.
- `internal/store/globaldb/global_db_loop_reconcile.go:16-61` + `internal/daemon/task_runtime_boot.go:389` — boot reconcile pattern + wiring point for the sweep.
- `internal/store/globaldb/schema/definitions/50_loops.sql` — loop tables incl. `loop_run_events:517-538` (timeline source), `loop_node_attempts:135` (attempts exposure), `loop_generation_outputs:76`.
- `internal/api/contract/loop_runs.go:96-258` + `loop_nodes.go:71-83` — payload families the new responses join.
- `internal/api/httpapi/loops_routes.go:35-80` + `internal/api/udsapi/loops_routes.go:38-89` — route registration (parity).
- `internal/api/core/loops.go:319-381` — SSE handler (unchanged; the accelerator).
- `internal/cli/loop.go:65-87`, `internal/cli/loop_nodes.go:11-44`, `internal/cli/task_list.go:13-89` — CLI verb homes.
- `internal/daemon/native_task_list.go:13-76` — native tool args.
- `internal/daemon/scheduler_loop_coordinator.go:13` — the `loop-coordinator` ref constant.
- `web/src/systems/tasks/lib/{task-formatters.ts,task-hierarchy.ts,task-catalog-filter.ts,tasks-list-filters.ts}` + `components/{task-card.tsx,task-subtask-list.tsx,tasks-list-surface.tsx,task-properties-rail.tsx}` — Tasks change/delete sites.
- `web/src/systems/loops/components/run-page/*` + `lib/{loop-node-lifecycle.ts,loop-run-progress.ts,loop-run-story.ts,loop-events.ts}` + `os/apps/loops/use-loop-run-page.ts:99-148` — run-page today-state.
- `web/src/systems/loops/components/detail/loop-body-dag.tsx:16-46` — static DAG (stays definition-only).
- `web/CLAUDE.md` — systems/query/SSE conventions binding the web work.

### Competitor References

- `.resources/smithers/apps/cli/src/monitor-ui/monitorModel.ts:506,685,285` — verdict cascade, event tiers, attempt sentences (briefing + timeline views).
- `.resources/smithers/packages/ui-core/src/runs/{runProgress,runEta,runHealth}.ts` — truthful-metrics types (`ratio: null`).
- `.resources/smithers/packages/gateway-ui/src/runNodeStatus.ts` — latest-execution-wins status merging (current-state truth).
- `.resources/mastra/packages/playground/src/domains/workflows/workflow/use-workflow-graph-runtime.tsx` — edge=data-flowed truth model (DAG).
- `.resources/mastra/packages/playground/src/domains/workflows/workflow/workflow-suspended-steps.tsx` — HITL card register (needs-you anatomy).
- `.resources/mastra/packages/playground/src/domains/workflows/context/workflow-run-provider.tsx` — stream + conditional snapshot poll reconciliation.
- `.resources/sim/apps/sim/app/workspace/[workspaceId]/logs/components/log-details/components/trace-view/trace-view.tsx` — roster gantt micro-bar, jump-to-error, leaf-most-failure selection.
- `.resources/sim/packages/workflow-renderer/src/edge/workflow-edge-view.tsx` — edge liveness + reduced-motion unmount.
- `.resources/sim/apps/sim/lib/logs/log-views.ts` — graduated agent read models with scan budgets.
- `.resources/temporal/common/persistence/sql/sqlplugin/sqlite/query_converter.go:151-205` — keyset pagination SQL (timeline/roster cursors).
- `.resources/temporal/service/history/api/describeworkflow/api.go:206-237` + `service/history/workflow/activity.go:92-215` — describe-vs-list split; stored-vs-derived per-node projection.
- `.resources/temporal/common/persistence/visibility/store/query/converter.go:187-210,486-488` — the mention-based opt-out trap our typed field avoids.

### Design and Analysis Sources

- `analysis/01_analysis_loop_kernel.md` — coupling map + the three settlement gaps (feeds ADR-006, phase 1).
- `analysis/02_analysis_api_surfaces.md` — surface/discriminator inventory (feeds ADR-004, phases 2-3).
- `analysis/03_analysis_web_ui.md` — web today-state + verdicts (feeds `_uiux.md`, phase 4).
- `analysis/04_analysis_design_intent.md` — institutional constraints (ADR-001..003 context).
- `analysis/05_analysis_market.md` — external patterns + citations (SD-012/L-036 corroboration).
- `analysis/06..09_analysis_ref_{mastra,smithers,sim,temporal}.md` — per-repo harvests behind the Competitor References above.
- `docs/design/opendesign/loop-legibility/` — landed visual contract set, binding for tasks 04/05. Six boards per `_uiux.md`: `docs/design/opendesign/loop-legibility/loop-legibility-tasks-list.html` (S1), `docs/design/opendesign/loop-legibility/loop-legibility-run-default.html` (S4), `docs/design/opendesign/loop-legibility/loop-legibility-needs-you.html` (S4 card), `docs/design/opendesign/loop-legibility/loop-legibility-run-dag.html` (S5 DAG), `docs/design/opendesign/loop-legibility/loop-legibility-run-roster.html` (S5 roster), `docs/design/opendesign/loop-legibility/loop-legibility-runs-roster.html` (S6). Companions: `docs/design/opendesign/loop-legibility/DESIGN-NOTES.md` (locked semantic contract), `docs/design/opendesign/loop-legibility/loop-legibility.css`, `docs/design/opendesign/loop-legibility/index.html` (set hub).

## Assumptions and Defaults

- Run-level usage (tokens, cost, budget %) is tracked by the runtime today (Usage rail + budget column exist); per-node usage renders only where session accounting provides it — otherwise absent (SD-007).
- Fork/rerun/time-travel always mint new run ids (cursor branch checks reduce to run identity) — verified against `run_forked`/rerun routes at implementation.
- `reconcile_interval` lands on the **existing** `[loops]` section (`internal/config/loops.go`, `LoopsConfigKey = "loops"`; lifecycle rule `loops.*` = restart-required already covers it) — verified against the codebase 2026-08-19; default `1m`; boot sweep unconditional.
- The notable/activity/chatter map is authored in code and exhaustive over the event-kind CHECK list — a unit test fails on any unclassified kind, so adding an event kind requires classifying it in the same change. Public `view=all` is the ordered union of all three tiers.
- CLI `--after` is a per-run sequence (scoped by the command's run argument); only HTTP page cursors are opaque run-bound tokens.
- Reveal filter state is ephemeral per navigation (US-002.AC-3); pagination limits default 50 (existing convention).
- Glossary gains **Loop cell** (operator term; plain register says "step"); grouped-entity human label is "Loop run".
- Cell task ids keep their current grammar (ids are stable identifiers; only their *exposure* as primary text changes).

## Architecture Decision Records

- [ADR-001: Loop execution records leave the Tasks surfaces by default](adrs/adr-001.md) — exclusion with typed reveal, 4-surface parity, loop-owned escalation lanes.
- [ADR-002: One loop run page, two registers via progressive disclosure](adrs/adr-002.md) — plain default read, operator depth one step deeper, no mode toggle.
- [ADR-003: Run-bound live DAG enters scope](adrs/adr-003.md) — reverses graph-eng deferral; read-only observability surface.
- [ADR-004: Classification rides existing provenance columns](adrs/adr-004.md) — no tasks schema change; kernel-neutral mapping at the handler layer.
- [ADR-005: Run reads are computed projections](adrs/adr-005.md) — on-demand roster, pure briefing, durable paged timeline; SSE accelerates.
- [ADR-006: Terminal settlement is part of the transition; sweep converges the rest](adrs/adr-006.md) — inline settlement everywhere + idempotent audited sweep + retroactive repair.
