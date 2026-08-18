# UI/UX Change Map: Loop Graph Completion (graph-eng)

Every UI surface this feature touches: where it lives today, what changes, which states must be designed, and the reference artboard each surface needs. Artboards land under `docs/design/opendesign/graph-eng/` and become the visual contracts the implementation tasks cite.

Companions: `_spec.md` Part I (behavior authority), `_user_stories.md` (states come from ACs/ECs), `_dx.md` (the non-UI half of the surface).

## Design constraints (apply to every artboard)

- Flat depth model, tokens from `packages/ui/src/tokens.css` only; no new tokens.
- Proposed signal mapping (final call belongs to the design pass): pending human request → **warning** on run-page surfaces, **danger** in bell contexts (the herdr attention vocabulary: needs-you = danger — ADR-006); expired request → **danger**; answered/amended → **info**; partial join → **warning**; canceled-by-strategy / route-not-taken / never-materialized → **neutral** (absence is calm, never alarming); fork lineage → **info**.
- Truthful UI: requests render only what the daemon persisted (prompt, redacted context preview + fetchable full redacted context, expected shape, deadlines) — nothing synthesized, and never the raw proposed-execution payload (previews only); a request whose run terminated shows the resolved outcome, never an answer form (US-007 EC-2); redacted context values render as redacted, not blank (US-007 EC-4); a partial run renders `partial` wherever its outcome appears — outcome card, list rows, diff — sourced from the run-level `completion_state`, never inferred client-side; wide fan-outs render aggregate counts, with the full set queryable, never 500 rows (US-017 EC-4).
- No optimistic updates on daemon-owned transitions (existing loops-system rule); answer forms disable on submit and reconcile from refreshed truth.
- Copy from `COPY.md` register; wait-kind sentence style follows the existing `WAIT_KIND_SENTENCE` map (`web/src/systems/loops/components/run-page/loop-run-parked-panels.tsx`).
- Keyboard-first: answer forms, decision buttons, and diff navigation fully keyboard-operable; no color-only signaling for partial/canceled (pair every chip with a label).

## Surface map

| #   | Surface                        | Kind    | Core change                                                        | Stories                        |
| --- | ------------------------------ | ------- | ------------------------------------------------------------------ | ------------------------------ |
| S1  | Run page · Needs-you card      | modify  | Carries ask/review requests with schema-driven answer forms        | US-001..003, 005..007          |
| S2  | Run page · story timeline      | modify  | New rows: request lifecycle, route taken, pruned, amended, forked  | US-011, US-018, US-004, US-022 |
| S3  | Run page · progress panel      | modify  | Strategy, threshold, partial state, window counts                  | US-012..015, US-017            |
| S4  | Run page · parked panels/rail  | modify  | Request wait kind + amend affordance on parked nodes               | US-004, US-006, US-007         |
| S5  | Run page · node row actions    | modify  | New verbs: amend, rerun-from-node                                  | US-004, US-021                 |
| S6  | Run page · inspect sheet       | modify  | Generation list gains diff + fork entry points; lineage panel      | US-019, US-020, US-022         |
| S7  | Run diff view                  | new     | Side-by-side generation/run comparison                             | US-019, US-020                 |
| S8  | Fork dialog                    | new     | Generation picker + pre-filled input override form                 | US-022                         |
| S9  | Loop-runs list + approvals bell| modify  | Pending-request badge/filter; bell includes loop requests          | US-007                         |
| S10 | Editor · palette + inspector   | modify  | `ask`/`route` nodes; strategy, iteration names, review block       | US-001, US-002, US-009, US-012..016 |
| S11 | Definition detail · body DAG   | modify  | Glyphs for ask/route; strategy summary on fan-out card             | US-009, US-012                 |

### S1. Run page · Needs-you card

- **Today**: `web/src/systems/loops/components/run-page/loop-run-needs-you-card.tsx:14-27` renders gate approvals only — the closed `approve / request_changes / reject` button set wired through `onDecision`.
- **Change**: the card becomes the request surface. Ask requests render prompt + author context + a schema-driven answer form derived from the request's expected shape (reusing the manual-wait payload machinery in `lib/loop-node-wait-payload.ts`). Review requests render the proposed arguments (original vs edited when editing) + the request's allowed decision set (`approve / edit / reject / respond`), each decision with its own affordance; `edit` opens the arguments editor pre-filled; `respond` opens an output form validated against the node's output shape. Gate approvals keep today's exact behavior.
- **States to design**: pending ask (US-001.AC-1); pending review with proposed args (US-002.AC-1); validation failure inline on submit (US-001.EC-1, field-level); already answered by someone else (US-001.EC-3 — resolved state replaces the form); expired (US-001.EC-4); multiple pending requests across fan-out lanes, each independently answerable and naming its lane (US-001.EC-7, US-007.EC-3); truncated context with marker (US-007.EC-1); redacted context values (US-007.EC-4); agent-permitted node showing responder rules (US-005).
- **Artboard**: `graph-eng-needs-you-requests.html`.

### S2. Run page · story timeline

- **Today**: `loop-run-story-timeline.tsx:23-32` renders event-derived story rows; row adapters in `lib/loop-run-story*.ts`; tone via `TONE_RING`.
- **Change**: new row families — request opened / answered (with actor + decision) / expired (with the route taken); route taken (selected route + matched condition or "default — no condition matched", per US-011.EC-1); branch pruned / canceled-by-strategy with the deciding cause (US-018.AC-2); node amended (actor + link to original); run forked (link to the fork); rerun generation opened (operator origin + reason).
- **States to design**: each row family in live and historical form; the aggregate row for high-width prune events (US-018.EC-2).
- **Artboard**: `graph-eng-timeline-rows.html`.

### S3. Run page · progress panel

- **Today**: progress panel renders the latest generation's fan-out counts (`loop-run-page-body.tsx:71-127` wires `progress: LoopRunProgressModel`).
- **Change**: strategy-aware progress: declared strategy + threshold; settled/active/pending/never-materialized counts (windowed width, US-017.AC-2); a partial completion state with coverage numbers that never renders as complete (US-013.AC-1/AC-3); canceled-by-strategy counts distinct from failures (US-012.AC-2).
- **States to design**: wait_all (today's view unchanged); fail_fast triggered (trigger branch named); best_effort met with partial badge; best_effort not met (failure path with coverage); race won (winner named, losers canceled); wide fan-out aggregates at 100× typical (US-017.EC-4); empty collection (US-013.EC-4).
- **Artboard**: `graph-eng-progress-strategies.html`.

### S4. Run page · parked panels / waits rail

- **Today**: `loop-run-parked-panels.tsx` + `loop-run-waits-rail.tsx` tally `timer | event | approval_escalation` waits with the `WAIT_KIND_SENTENCE` copy map.
- **Change**: a request wait kind joins the map (ask: "is waiting for an answer"; review: "is waiting for a decision on its proposed action"), each linking to its S1 form; the rail's counts include pending requests.
- **States to design**: request wait row (pending / escalated / near-expiry); zero state stays a plain `0`.
- **Artboard**: covered by `graph-eng-needs-you-requests.html` (rail column).

### S5. Run page · node row actions

- **Today**: `loop-node-row-actions.tsx` + `loop-node-control-dialog.tsx:36-45` offer pause/resume/cancel/kill/requeue with closed-mode copy maps.
- **Change**: `Amend output` on parked/paused nodes with a declared output shape (dialog: schema-validated editor, original shown read-only, reason field; absent — not disabled — when the node has no shape, per truthful-UI); `Rerun from here` on settled nodes (dialog: rerun-set preview — the node + transitive dependents — and reason field; absent while the node is parked or the run is mid-generation, US-021.EC-1/EC-2).
- **States to design**: amend dialog (edit / validation failure / conflict on concurrent amend US-004.EC-3); rerun dialog with rerun-set preview; both verbs' deterministic daemon answers rendered as information (existing `LoopControlAnswer` pattern).
- **Artboard**: `graph-eng-node-verbs.html`.

### S6. Run page · inspect sheet

- **Today**: `loop-run-inspect-sheet.tsx:35-45` renders generations, verdicts, resolved runtimes, frames.
- **Change**: each generation row gains `Compare…` (opens S7 seeded with that generation) and `Fork from here` (opens S8); a lineage block renders `forked_from` and `forks[]` with two-way links (US-020.AC-2, US-022.AC-2); route causes and strategy settlements appear in the generation detail.
- **States to design**: lineage present/absent; fork chain (fork of a fork, US-022.EC-2).
- **Artboard**: `graph-eng-inspect-lineage.html`.

### S7. Run diff view (new)

- **Change**: a full-height comparison surface launched from S6 (and deep-linkable): generation↔generation within a run, or run↔run for the same loop. Per-node rows grouped by change kind (`changed / rerun / skipped / carried / verdict`), matching the CLI diff vocabulary in `_dx.md`; unchanged summarized, large outputs summarized with sizes/hashes and a link to full content (US-019.AC-2, EC-3); input diff block for run↔run (US-020.AC-1); definition-divergence banner when the two runs pin different versions (US-020.EC-1); live-side label when one run is still executing (US-020.EC-3).
- **States to design**: empty diff (US-019.EC-1); carried-forward marking (US-019.EC-2); cross-loop rejection is CLI/API-only (the UI never offers it — pickers are same-loop scoped).
- **Artboard**: `graph-eng-run-diff.html`.

### S8. Fork dialog (new)

- **Change**: generation picker (defaulting to the inspected one) + the loop's declared-input form pre-filled with the source run's inputs (reusing the run-form machinery behind `use-loop-run-form.ts`); submit starts the fork and navigates to the new run; validation errors render exactly as the run form's (US-022.AC-3).
- **States to design**: no overrides (pure replay, US-022.EC-4); validation failure; source content unavailable (US-022.EC-1 — submit blocked with the deterministic reason).
- **Artboard**: `graph-eng-fork-dialog.html`.

### S9. Loop-runs list + attention bell (rides the herdr-parity pipeline — ADR-006)

- **Today**: loop-runs list surfaces run status. The bell is being rebuilt by the herdr-parity program (in flight): **Needs you / Finished** sections, workspace-labeled rows, badge + title fed by a daemon-computed session summary, jump = land on the source (herdr `_uiux.md` S3; `web/src/systems/os/components/attention-bell.tsx`, `use-os-attention.ts`, `attention-model.ts`). herdr's Non-Goals explicitly leave task/loop attention semantics out (herdr `_spec.md:183`) — task approval rows already sit in the bell as the precedent pattern.
- **Change**: loop requests join the redesigned bell as rows in **Needs you**, following the task-approval-row pattern: workspace label, request kind + loop name + age, jump lands on the run page's request form (S1). The badge/title count adds the loops-owned `aggregates.pending` projection (from `_dx.md`'s requests inventory) to herdr's session `needs_you` count — composition of two daemon-computed counts, never a row-page count, per-workspace error isolation preserved. The loop-runs list gains a needs-you indicator on runs with pending requests. **Concrete seam (post-merge):** this spec edits only the `web/src/systems/os` composition modules — `use-os-attention.ts` gains a loops request source, and the bell row model receives loop rows — and never touches the session-attention contracts (the `attention-summary` endpoint/consumption, session badge derivation in `attention-model.ts`, `session_attention_changed` handling, the notifier). This surface lands after herdr-parity merges (a global precondition of the spec), integrating with the merged files as they then exist.
- **States to design**: loop-request row in Needs you (pending / near-expiry); resolved/expired request never renders as a live row (it leaves the section, mirroring herdr's no-dismiss inbox-drain behavior); list-row indicator; stale/disconnected states inherit the herdr bell's existing treatment unchanged.
- **Artboard**: `graph-eng-bell-requests.html` — **iterates on `docs/design/opendesign/herdr-parity/herdr-parity-bell.html` §01 with `herdr.css` grammar** (never a parallel bell design): adds only the loop-request row kind. In bell contexts the row follows herdr's signal vocabulary (needs-you = danger); the loops-domain warning mapping applies only inside the run page.

### S10. Editor · palette + inspector

- **Today**: `loop-editor-canvas.tsx:34-53` + palette/inspector parts (`loop-editor-palette.tsx`, `loop-editor-inspector.tsx`, field parts); linter dock renders shared-linter diagnostics; `isValidConnection` is a UX hint only (the Go linter owns invariants).
- **Change**: palette gains `ask` (control) and `route` (control); inspector gains — ask: prompt/context/expect/responders/expires fields (expect editor reuses the JSON-schema field pattern from `loop-editor-json-field.tsx`); route: ordered routes list (condition + target picker constrained to forward nodes) + required default picker; fan-out: strategy field (kind, threshold, `missing: acceptable` acknowledgment), `bind_as`/`index_as` names; action nodes: review block (decisions, `when`, prompt, responders, on_reject route, expires); condition-bearing nodes: `on_eval_error` override. Linter dock surfaces the new codes (`route_default_missing`, `route_target_invalid`, `strategy_coverage_undeclared`, `strategy_threshold_invalid`, `iteration_name_conflict`, `unknown_parameter` for the removed field).
- **States to design**: each new inspector section (valid / lint-error); route list reordering (declaration order is the tiebreak, US-009.EC-1); DSL view round-trip of every new field (the bijective codec keeps Graph/DSL toggle lossless).
- **Artboard**: `graph-eng-editor-route-ask.html`.

### S11. Definition detail · body DAG

- **Today**: `loop-body-dag.tsx:12-15` — horizontal spine of neutral node cards, class glyph carries identity, color is state-only; `fanOutSummary` from `lib/loop-graph.ts`.
- **Change**: glyphs for `ask` and `route`; fan-out card summary includes strategy + iteration names; route card lists route count + default.
- **States to design**: read-only rendering of the new kinds; run-page DAG stays out of scope (timeline remains the run view — the live topology view is an explicit non-goal of this spec).
- **Artboard**: covered by `graph-eng-editor-route-ask.html` (detail column).

## Component plan (design → production mapping)

### Rules

- Every request form derives from the daemon-persisted expected shape — no hand-authored per-loop forms; the manual-wait payload editor is the base.
- Decision buttons render only the request's persisted `decisions` set — an absent decision is absent, not disabled.
- Diff rows are data, not prose: one row per node change, grouped by change kind, matching the CLI vocabulary 1:1.
- All new mutations follow the existing invalidation sets (`runDetail`, `runsByWorkspace`, `nodeInventoryByWorkspace`) plus a new `requestsByWorkspace` key; no optimistic paints.

### New `@compozy/ui` primitives

None expected — forms, sheets, dialogs, badges, tables, and diff-row layout compose from the existing inventory (`packages/ui/src/index.ts`); the design pass must justify any exception against it.

### New domain components (`web/src/systems/loops/`)

| Component | Composed from | Used by |
| --- | --- | --- |
| `LoopRequestAnswerForm` | schema-driven field renderer (wait-payload machinery) + form primitives | S1, S4 |
| `LoopRequestDecisionBar` | button group + note field | S1 |
| `LoopReviewProposedArgs` | key-value table + editable JSON field | S1 |
| `LoopRunDiffView` + `LoopRunDiffRow` | table/list primitives + code/hash chips | S7 |
| `LoopForkDialog` | dialog + run-form input fields | S6, S8 |
| `LoopNodeAmendDialog` | dialog + schema-validated editor | S5 |
| `LoopNodeRerunDialog` | dialog + rerun-set preview list | S5 |
| `LoopStrategyProgress` | stat row + chips | S3 |
| Editor field parts: `loop-editor-route.tsx`, `loop-editor-ask.tsx`, `loop-editor-strategy-field.tsx`, `loop-editor-review-field.tsx` | existing editor field patterns | S10 |

### Signal & state mapping

| Design state | Primitive + token |
| --- | --- |
| Pending request chip | existing status chip + warning token |
| Expired request | chip + danger token |
| Answered / amended provenance line | inline meta text + info token |
| Partial join badge | chip + warning token, always with the word "partial" and coverage numbers |
| Canceled-by-strategy / not-taken / never-materialized | neutral chip, cause text from the event payload |
| Fork lineage link | link row + info token |
| Diff change kinds | `changed` accent · `rerun` info · `skipped` neutral · `carried` muted · `verdict` per existing verdict tones |
```
