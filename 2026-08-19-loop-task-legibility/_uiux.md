# UI/UX Change Map: Loop & Task Legibility

Every UI surface this feature touches: where it lives today, what changes, which states must be designed, and the reference artboard each surface needs. Artboards are landed under `docs/design/opendesign/loop-legibility/` and are the visual contracts the implementation tasks cite by full path. A UI-bearing task whose cited board is missing at execution time blocks; it never improvises the reference.

Companions: `_spec.md` Part I (behavior authority), `_user_stories.md` (states come from ACs/ECs), `_dx.md` (the non-UI half of the surface).

## Design constraints (apply to every artboard)

- **Default read discipline (SD-012 / L-036):** every surface below declares its default read; machine ids, raw enums, and mechanism copy render only in disclosed operator views. Failure and needs-you signals never collapse. Status copy is meaning ("second reviewer rejected the draft"), never mechanics ("node_failed g2") — explicit `COPY.md` label map, no title-cased enums.
- **Signal palette proposal (final call: design pass):** needs-you/failed → danger `#E0635A`; retrying/waiting/paused → warning `#D6A647`; succeeded → success `#5FBF85`; terminal-done check + neutral informational chips → info `#8E8EB5`; the single live "running" accent → action `#E8572A`. State is never color-alone — icon + text pair every chip (WCAG floor). Canceled-by-strategy / route-not-taken / never-materialized → neutral ramp (absence is calm — graph-eng grammar, kept).
- **Incumbent grammar (binding):** the run page already wears the graph-eng set's locked request grammar, shipped to production — `docs/design/opendesign/graph-eng/DESIGN-NOTES.md` + `docs/design/opendesign/graph-eng/graph-eng.css` + `docs/design/opendesign/graph-eng/graph-eng-needs-you-requests.html`: pending human request = **warning** on run-page surfaces (danger reserved for bell contexts), near-expiry = warning + countdown, expired = danger, answered/amended/fork lineage = info, partial = warning + the literal word `partial` with coverage numbers, decisions vocabulary exactly `approve / edit / reject / respond`, wait-kind sentences locked. The palette proposal above resolves **inside** that lock: needs-you pending inks warning-family on the page (danger stays for failed/quarantined states and the bell re-ink), and the resolution is recorded in `docs/design/opendesign/loop-legibility/DESIGN-NOTES.md`. Iterate the graph-eng needs-you board grammar — never fork a second request vocabulary on the same page.
- **Structure vs status on separate channels** (mastra pattern): node *kind* (agent/gate/collect/route) renders as glyph/border from the neutral ramp; *status* renders as signal icon+text. The two never share a channel. The node-kind glyph inventory is authoritative and **additive** from `web/src/systems/loops/lib/loop-palette.ts` + the shipped detail DAG (`components/detail/loop-body-dag.tsx`) — boards may stage a subset as story, but never hide or drop a kind as an inventory statement (graph-eng lesson).
- **Attempts are metadata, never siblings** (inngest/smithers): "attempt 2" chips on the step/node row; per-attempt history in disclosure. Recovered nodes read by their current state (current-state truth rule).
- **Rollups derived, never entered**; fan-out renders as counts ("7/10 · 1 failed"), never per-item elements (US-011.EC-1).
- **Truthful UI:** unknown renders as unknown; "no detail exists" is a designed state (vercel pattern); transport-degraded states (connecting/offline) never render as "nothing here" (smithers `runsLandingState`).
- Reuse before create: compose from `@compozy/ui` (`packages/ui/src/index.ts`); domain composites in `web/src/systems/{tasks,loops}/`. Copy sources: `COPY.md`; glossary terms (`loop cell` operator-only; plain register says "step"; grouped entity label "Loop run").

## Surface map

| #  | Surface | Kind | Core change | Stories |
| -- | ------- | ---- | ----------- | ------- |
| S1 | Tasks list + kanban | modify | Loop records excluded by default; hidden reveal filter; nesting machinery deleted | US-001, US-002 |
| S2 | Tasks dashboard + inbox | modify | Aggregates/lanes exclude loop records (no redesign) | US-003 |
| S3 | Task detail (revealed loop records) | modify | Loop provenance block + link back to run page | US-002, US-015 |
| S4 | Loop run page — default read | modify (redesign) | Briefing verdict lead, steps+rounds progress, outcome-first terminal, narrated durable story | US-005..009 |
| S5 | Loop run page — operator register | new (within page) | Live run DAG, complete node roster, generation history, per-node attempt history, raw events | US-011..015 |
| S6 | Loop runs roster (`/loop-runs`) | modify | Plain-words outcome leads; needs-you runs first and distinct | US-010 |
| S7 | Loop node inventory | modify (light) | Unchanged contract; rows keep deep-links; no new states | US-014 (reach) |

### S1. Tasks list + kanban

- **Today**: every coordinator/cell renders as a row; client-side nesting only when the coordinator shares the loaded page (`web/src/systems/tasks/lib/task-hierarchy.ts:17-35`); id-regex identity (`lib/task-formatters.ts:73-111`); subtask disclosure (`components/task-subtask-list.tsx:78-114`); incoherent bucket counts (`components/tasks-list-surface.tsx:44-47`); toolbar filters = status/priority/owner only (`lib/tasks-list-filters.ts:80-114`).
- **Change**: loop records leave the default listing (server-owned exclusion). A quiet filter control (hidden by default, off on every navigation — US-002.AC-3) reveals them as visually distinguished rows: loop glyph, plain identity ("revisao-paralela · run" / "revisao-paralela · round 1 · step revisor-perf"), activation → run page. Delete: `parseLoopTaskId`/regex, `buildTaskListTree` loop nesting, `task-subtask-list` loop summary path. Bucket counts become coherent by construction.
- **Default read**: "what work do I have, what state is each item in, what needs me" — work items only; zero loop rows; no mono ids beyond existing task ids in secondary text. Demoted to reveal: all loop execution records.
- **States to design**: default with active loop (US-001.AC-1/2); revealed with mixed records (US-002.AC-1); revealed-empty ("no loop records in this workspace" — US-002.EC-1); revealed row whose run was retention-deleted (US-002.EC-2); workspace with only loop records → true empty state (US-001.EC-2); kanban default (roots = work items only).
- **Artboard**: `docs/design/opendesign/loop-legibility/loop-legibility-tasks-list.html`.

### S2. Tasks dashboard + inbox

- **Today**: server aggregates and inbox lanes include loop cells (escalated cell lands beside human work — `analysis/03_analysis_web_ui.md` §Verdict 1.6).
- **Change**: aggregates, facets, and lanes exclude loop records (server-side, same default as S1). No layout redesign (Non-Goal). Loop escalations surface only through the attention bell's loop lane and the run page.
- **Default read**: unchanged questions ("how much work, in what states, what's in my inbox") — now answered over work items only.
- **States to design**: none new — numbers change, composition does not. QA asserts count coherence (US-003.AC-1/2, EC-1/2).
- **Artboard**: none (no visual change; covered by scenario assertions).

### S3. Task detail (revealed loop records)

- **Today**: `created_by.ref` as plain text, no run link, no metadata rendering (`components/task-properties-rail.tsx:97,270`).
- **Change**: coordinator/cell task detail gains a loop provenance block — loop name, run link ("Open run"), round, step, item — in plain words; the record is labeled as a loop execution record. Link back is mandatory (US-015.AC-2).
- **Default read**: "what is this record, which run owns it, where do I go to act" — the run page is the action home; this page is the mechanism view (operator register by nature).
- **States to design**: cell with attempt history; coordinator; record whose run is gone (truthful degrade — US-002.EC-2).
- **Artboard**: reuse S1's revealed-row grammar; no dedicated artboard.

### S4. Loop run page — default read (redesign)

- **Today**: 9+ competing sections + 4-card rail (`components/run-page/loop-run-page-body.tsx`); exception-only node rows (`lib/loop-node-lifecycle.ts:195`); progress only for widest fan-out node (`lib/loop-run-progress.ts:153-201`); story from a 500-frame client buffer without durable backfill (`lib/loop-events.ts:139-152`, `use-loop-run-page.ts:116-123`); task-run link hero-only/live-only (`loop-run-now-card.tsx:258-268`).
- **Change**: default read becomes four elements, in order: (1) **briefing strip** — the `/briefing` verdict: tone + plain headline + detail + inline unblock action (smithers `diagnoseRun` register); (2) **needs-you** — card anatomy: what is asked, who asks, choices, expiry; never collapses; multiple → ordered list with count; (3) **progress** — "step 3 of 6 · round 2" with fan-out rollup chips and attempt metadata; single-pass runs hide the round counter; (4) **story** — narrated beats from the durable paged timeline (titles in meaning; heartbeat-class events coalesced; `notable` view default), with outcome + produced artifacts leading once terminal (partial results labeled). Everything else demotes to the operator register (S5) behind one disclosure ("Inspect"). Rail keeps **Usage** — tokens · cost · budget consumption · round count · duration — always visible as part of the default read (the briefing test's "spend"), plus About; the rest of today's rail demotes.
- **Default read**: "what is running · what needs me · how far along · what has it spent · what came out" — no machine ids, no raw enums, no panel competition. Demoted: DAG, roster, generations, raw events, attempts detail, resolved runtimes, waits detail.
- **States to design**: running-healthy (US-005.AC-1/2); needs-you single + multiple (US-007.AC-1..3, EC-1); queued/admission-parked with reason (US-005.EC-1); watching/dormant (US-005.EC-2); terminal done with artifacts (US-008.AC-1); terminal failed — failure signal outside any accordion (US-008.AC-2); terminal no-op / canceled with actor (US-008.EC-1/2); partial outputs (US-008.AC-3); pruned blob (US-008.EC-3); long-run story paging (US-009.AC-2, EC-1); fork/time-travel beat (US-009.EC-3); budget nearly-exhausted / exhausted (warning tone — truthful to the runtime budget model); reduced-motion.
- **Artboard**: `docs/design/opendesign/loop-legibility/loop-legibility-run-default.html` (+ `docs/design/opendesign/loop-legibility/loop-legibility-needs-you.html` for the card anatomy).

### S5. Loop run page — operator register

- **Today**: Inspect sheet = generation stubs + capped raw events (`run-page/loop-run-inspect-sections.tsx:211-312`); no DAG bound to a run (`components/detail/loop-body-dag.tsx:16-46` is static); no roster of healthy nodes; attempts partial (`lib/loop-node-lifecycle.ts:118-136`).
- **Change**: one disclosure level below S4, three coordinated views over the run-roster read model (`/nodes`):
  - **Live run DAG** (ADR-003): authored topology with per-node status (icon+text chip), fan-out rollup counts on the node, edge lighting = data flowed (taken path lights; **`not_taken` only on durable route evidence, neutral-dim; reachable-but-unmaterialized renders `pending` — the two are visually and semantically distinct**), liveness carried by edges (pulse toward the active node — sim pattern), auto-center on whatever needs a human; node click → node panel (state, attempts history, next-retry time, timing, error class and cancellation cause/actor in plain words + "Open session" / "Open record" / "View child run" — all valid post-terminal); node verbs (existing pause/resume/cancel/kill/requeue/amend) from the node panel (US-014).
  - **Node roster**: every node × round, healthy included; columns state/attempt/duration/tokens·cost/session; gantt micro-bar per row (sim); per-attempt history disclosure; round filter; fan-out grouped under the node with rollup (US-012).
  - **Generation history**: per-round outcomes, scores/verdicts when the loop defines them, per-round usage (tokens · cost), node results; Compare/Fork preserved (US-013); crash-interrupted rounds render their true partial state.
  - Raw events keep the `all` view escape hatch (paged from the durable timeline).
- **Default read**: n/a — this *is* the disclosed depth; its own internal lead is the DAG ("where is the run in its shape").
- **States to design**: DAG running/terminal/failed/quarantined node; **pending (reachable, upstream unsettled) as a distinct calm state from not-taken (durable route evidence)**; wide fan-out (rollup only — US-011.EC-1); deep graph navigation (US-011.EC-3); not-taken branch (US-011.EC-2); roster with 10-attempt node incl. next-retry time (US-012.AC-2); strategy-canceled vs operator-canceled with cause/actor (US-012.EC-2); zero-action-node run (US-012.EC-1); concurrent-intervention conflict (US-014.EC-1/2); stale-view rejection; pruned session link degrade (US-015.EC-1); reduced-motion (edge pulse unmounts).
- **Artboard**: `docs/design/opendesign/loop-legibility/loop-legibility-run-dag.html`, `docs/design/opendesign/loop-legibility/loop-legibility-run-roster.html`.

### S6. Loop runs roster (`/loop-runs`)

- **Today**: KPIs + Active/Past tables (Outcome·Loop·Inputs·Gens·Best·Started·Budget — `components/runs/loop-runs-table.tsx`), pending-request badges; mono run ids prominent.
- **Change**: rows lead with plain outcome/status + needs-you marker; needs-you group sorts first, then active, then terminal (smithers `GROUP_ORDER`) — **server-owned: ordering, attention summary, and step/round progress come from the extended runs-list read applied before pagination** (`_dx.md` `compozy loop runs`; never client-side page sorting or N+1 run reads). Run id demotes to secondary text; row → run page. Columns re-ranked: Loop · Status/needs-you · Progress (steps/round) · Started · Duration; Gens/Best/Budget demote to the run page.
- **Default read**: "which runs need me · which are moving · which finished" (US-010.AC-1/2).
- **States to design**: needs-you row distinct; empty roster (US-010.EC-1); dozens-active scale (US-010.EC-2); transport-degraded (connecting ≠ empty).
- **Artboard**: `docs/design/opendesign/loop-legibility/loop-legibility-runs-roster.html`.

### S7. Loop node inventory

- **Today**: four exception states, server-ordered, loop/run filters (`components/runs/loop-node-inventory-view.tsx`; contract `docs/qa/scenarios/LP-web-detail-inventory-contract.md`).
- **Change**: contract unchanged (workspace scope keeps exception states — the full roster is run-scoped in S5). Rows keep deep-links to run page/nodes. Only vocabulary alignment ("step" plain labels).
- **Default read**: unchanged ("which nodes across my workspace are parked and why").
- **States to design**: none new.
- **Artboard**: none.

## Component plan (design → production mapping)

### Rules

- All run-page views bind to the run-roster read (`/nodes`) + durable timeline (`/timeline`) + briefing (`/briefing`) — one source, several projections; no view derives node state from SSE frames alone (SSE accelerates, reads reconcile).
- The DAG is a read-only observability surface: no editor affordances; renderer internals may be shared with the editor canvas only if authoring chrome is fully absent (Part II decision).
- Delete targets in `web/`: `lib/task-formatters.ts` loop regex + `taskShortId` loop branch; `lib/task-hierarchy.ts` loop nesting + `subtaskSummaryLabel`; `components/task-subtask-list.tsx` (loop path); story reliance on `MAX_STORY_FRAMES` buffer as sole history.

### New `@compozy/ui` primitives

None expected — verdict strip, chips, progress, tables, disclosures compose from existing primitives (`Badge`, `Card`, progress and table primitives per `packages/ui/src/index.ts`); the DAG canvas is domain-specific. Any genuinely generic gap found during design (e.g., a generic `StatusChip` variant) lands in `packages/ui` with story + test — justified against the inventory first.

### New domain components

Per `web/src/systems/loops/`: `LoopRunBriefing` (strip; consumes briefing), `LoopRunNeedsYou` (card list; consumes requests/approvals), `LoopRunStepsProgress` (steps+rounds; consumes roster rollups), `LoopRunStory` (narrated beats; consumes timeline paged), `LoopRunDag` (live graph; consumes roster + definition snapshot), `LoopNodeRoster` (table + gantt micro-bar), `LoopNodePanel` (node detail + verbs), `LoopGenerationHistory`. Per `web/src/systems/tasks/`: `TaskLoopProvenance` (detail block), revealed-row loop badge variant (`TasksListRow` extension, domain-prefixed).

### Signal & state mapping

Covers the FULL closed roster projection enum (`_spec.md` roster contract: 14 persisted output states + derived `not_taken`) — every chip the DAG/roster/briefing can render maps here; the needs-you row obeys the incumbent-grammar lock above.

| Design state | Tone family + primitive |
| --- | --- |
| failed / quarantined chip | danger — badge + icon + literal state word |
| needs-you (pending request/approval on the page) | warning per the graph-eng lock; danger only as the bell re-ink |
| retrying / waiting / paused / awaiting_child / control_pending / awaiting_goal | warning family — parked-on-something; each chip carries its literal state word (retrying adds `next_retry_at`) |
| partial | warning + the literal word `partial` with coverage numbers (graph-eng lock) |
| succeeded chip | badge + success |
| running (single live accent) | badge/pulse + action; edge pulse motion token, unmounted under reduced-motion |
| terminal-done check / answered / amended / fork lineage / neutral info | info |
| canceled (strategy or operator) | neutral ramp — disposition/cause/actor carried by text, never an alarm color (UT-005: canceled reads neutral) |
| pending / queued (reachable, not started) | neutral ramp, hollow/outline glyph + literal word — MUST stay visually and semantically distinct from `not_taken` (Safety Invariant 14) |
| not-taken / never-materialized / strategy-parked branch | neutral ramp dim + route glyph (`git-branch`/`minus`/`circle-dashed` per graph-eng), no signal color |

## Canonical data story (all six artboards)

One coherent dataset across the set — the `_dx.md` transcripts are the authority; boards render these entities, never invented ones:

- Loop `revisao-paralela`, topology `implementar` → fan-out `revisores` ×3 (`revisor-seguranca`, `revisor-perf`, `revisor-estilo`) → `sintetizador` → `saida`; approval gate `aplicar-correcoes`.
- Live run `looprun-8f3ab2c1d4e5f607`: round 1, step 4/6, approval waiting 3m on `revisor-perf`'s gate, `revisor-estilo` recovered on attempt 2 (attempt 1 `tool_error`), usage 82.4k tokens · $0.31 · 12% budget · 9m40s.
- Terminal run `looprun-77aa01b2c3d4e5f6`: done after 2 rounds (18m12s), 214.5k tokens · $0.87 · 38% budget, artifact `post-final.md` via output `saida`.
- Second loop `fabrica-assistida` (running, step 2/9, started 18:41) fills rosters/lists; terminal/canceled/failed roster rows derive from these two loops' history.
- Degraded/edge states reuse the same entities (the retention-deleted record keeps its `looprun-…` id with `loop_name` absent; the pruned artifact keeps the name `post-final.md`).
- Run/task ids render only where the operator register allows them — never as primary text in default reads.

## Set deliverables (OD conventions)

Beyond the six boards, the set ships: `docs/design/opendesign/loop-legibility/loop-legibility.css` (chaptered, append-only after the marked append point — later runs never restyle earlier chapters), `docs/design/opendesign/loop-legibility/DESIGN-NOTES.md` (the locked semantic contract; records the needs-you tone resolution inside the graph-eng lock), and `docs/design/opendesign/loop-legibility/index.html` (set hub). CSS authority chain (highest first): `packages/ui/src/tokens.css` + production run-page seams (`loop-run-page-body.tsx` et al.) > this `_uiux.md` (+ `_user_stories.md` ACs/ECs) > `docs/design/opendesign/design-system/ds-core.css` + `ds-shell.css` (+ the `docs/design/opendesign/graph-eng/graph-eng.css` request-grammar chapters) > the set css. Each board = final surface + states lab; labs run full-viewport, staged page fragments render at production content width (fluid ≤1240) with the 320px rail pixel-true (graph-eng lab-layout precedent).
