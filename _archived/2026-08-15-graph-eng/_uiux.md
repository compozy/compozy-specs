# UI/UX Change Map: Loop Graph Completion (graph-eng)

Every UI surface this feature touches: where it lives today, what changes, which states must be designed, and the reference artboard each surface binds to. Visual contracts are delivered under `docs/design/opendesign/graph-eng/`: `DESIGN-NOTES.md` (locked semantics + data story), `index.html` (set hub), and the ten `graph-eng-*.html` boards. Implementation tasks cite those files as Visual Contract rows; artboard CSS is a contract, never a stylesheet to import.

Companions: `_spec.md` Part I (behavior authority), `_user_stories.md` (states come from ACs/ECs), `_dx.md` (the non-UI half of the surface).

## Design constraints (apply to every artboard)

- Flat depth model, tokens from `packages/ui/src/tokens.css` only; no new tokens.
- Signal mapping (locked in `docs/design/opendesign/graph-eng/DESIGN-NOTES.md`): pending human request → **warning** on run-page surfaces, **danger** in bell contexts (the herdr attention vocabulary: needs-you = danger — ADR-006); expired request → **danger**; answered/amended → **info**; partial join → **warning**; canceled-by-strategy / route-not-taken / never-materialized → **neutral** (absence is calm, never alarming); fork lineage → **info**.
- Truthful UI: requests render only what the daemon persisted (prompt, redacted context preview + fetchable full redacted context, expected shape, deadlines) — nothing synthesized, and never the raw proposed-execution payload (previews only); a request whose run terminated shows the resolved outcome, never an answer form (US-007 EC-2); redacted context values render as redacted, not blank (US-007 EC-4); a partial run renders `partial` wherever its outcome appears — outcome card, list rows, diff — sourced from the run-level `completion_state`, never inferred client-side; wide fan-outs render aggregate counts, with the full set queryable, never 500 rows (US-017 EC-4).
- No optimistic updates on daemon-owned transitions (existing loops-system rule); answer forms disable on submit and reconcile from refreshed truth.
- Copy from `COPY.md` register; wait-kind sentence style follows the existing `WAIT_KIND_SENTENCE` map (`web/src/systems/loops/components/run-page/loop-run-parked-panels.tsx`).
- Keyboard-first: answer forms, decision buttons, and diff navigation fully keyboard-operable; no color-only signaling for partial/canceled (pair every chip with a label).
- Editor chrome follows the calm-defaults posture: optional rails ship collapsed by default and open contextually (the inspector auto-opens on selection); every chrome state — rail collapse, inspector width, dock folds — persists per user, and full-bleed canvas wins (S12).
- Artboard canvas content is the set's locked illustrative data story (`release-train` — `docs/design/opendesign/graph-eng/DESIGN-NOTES.md`), never an inventory statement: the editor palette is **additive** — every existing node kind stays and `ask`/`route` join them; runtime truth (`web/src/systems/loops/lib/loop-palette.ts` + the Go DSL) owns the kind inventory.

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
| S12 | Editor · chrome & ergonomics   | modify  | Collapsible rails, undo/redo, quick-add, clipboard, edge/shortcut ergonomics | Addendum 2026-08-16 (no US)    |

**Competitive baseline (2026-08-16):** `analysis/sim-uiux.md` audits every surface against Sim (`.resources/sim`) and the reference boards in `docs/design/opendesign/graph-eng/`. Verdict: the request/strategy/time-travel surfaces meet or beat Sim — pending human input is invisible inside Sim's editor, fan-out has no progress surface, and amend/fork/run-data-diff have no equivalent — while the one real deficit is editor chrome ergonomics, closed by S12 (plus the C13 route-row pattern folded into S10).

### S1. Run page · Needs-you card

- **Today**: `web/src/systems/loops/components/run-page/loop-run-needs-you-card.tsx:14-27` renders gate approvals only — the closed `approve / request_changes / reject` button set wired through `onDecision`.
- **Change**: the card becomes the request surface. Ask requests render prompt + author context + a schema-driven answer form derived from the request's expected shape (reusing the manual-wait payload machinery in `lib/loop-node-wait-payload.ts`). Review requests render the proposed arguments (original vs edited when editing) + the request's allowed decision set (`approve / edit / reject / respond`), each decision with its own affordance; `edit` opens the arguments editor pre-filled; `respond` opens an output form validated against the node's output shape. Gate approvals keep today's exact behavior.
- **States to design**: pending ask (US-001.AC-1); pending review with proposed args (US-002.AC-1); validation failure inline on submit (US-001.EC-1, field-level); already answered by someone else (US-001.EC-3 — resolved state replaces the form); expired (US-001.EC-4); near-expiry (warning meta + deadline countdown, matching S4's rail state); multiple pending requests across fan-out lanes, each independently answerable and naming its lane (US-001.EC-7, US-007.EC-3); truncated context with marker (US-007.EC-1); redacted context values (US-007.EC-4); agent-permitted node showing responder rules (US-005).
- **Artboard**: `docs/design/opendesign/graph-eng/graph-eng-needs-you-requests.html`.

### S2. Run page · story timeline

- **Today**: `loop-run-story-timeline.tsx:23-32` renders event-derived story rows; row adapters in `lib/loop-run-story*.ts`; tone via `TONE_RING`.
- **Change**: new row families — request opened / answered (with actor + decision) / expired (with the route taken); route taken (selected route + matched condition or "default — no condition matched", per US-011.EC-1); branch pruned / canceled-by-strategy with the deciding cause (US-018.AC-2); node amended (actor + link to original); run forked (link to the fork); rerun generation opened (operator origin + reason).
- **States to design**: each row family in live and historical form; the aggregate row for high-width prune events (US-018.EC-2).
- **Artboard**: `docs/design/opendesign/graph-eng/graph-eng-timeline-rows.html`.

### S3. Run page · progress panel

- **Today**: progress panel renders the latest generation's fan-out counts (`loop-run-page-body.tsx:71-127` wires `progress: LoopRunProgressModel`).
- **Change**: strategy-aware progress: declared strategy + threshold; settled/active/pending/never-materialized counts (windowed width, US-017.AC-2); a partial completion state with coverage numbers that never renders as complete (US-013.AC-1/AC-3); canceled-by-strategy counts distinct from failures (US-012.AC-2).
- **States to design**: wait_all (today's view unchanged); fail_fast triggered (trigger branch named); best_effort met with partial badge; best_effort not met (failure path with coverage); race won (winner named, losers canceled); wide fan-out aggregates at 100× typical (US-017.EC-4); empty collection (US-013.EC-4).
- **Artboard**: `docs/design/opendesign/graph-eng/graph-eng-progress-strategies.html`.

### S4. Run page · parked panels / waits rail

- **Today**: `loop-run-parked-panels.tsx` + `loop-run-waits-rail.tsx` tally `timer | event | approval_escalation` waits with the `WAIT_KIND_SENTENCE` copy map.
- **Change**: a request wait kind joins the map (ask: "is waiting for an answer"; review: "is waiting for a decision on its proposed action"), each linking to its S1 form; the rail's counts include pending requests.
- **States to design**: request wait row (pending / escalated / near-expiry); zero state stays a plain `0`.
- **Artboard**: covered by `docs/design/opendesign/graph-eng/graph-eng-needs-you-requests.html` (rail column).

### S5. Run page · node row actions

- **Today**: `loop-node-row-actions.tsx` + `loop-node-control-dialog.tsx:36-45` offer pause/resume/cancel/kill/requeue with closed-mode copy maps.
- **Change**: `Amend output` on parked/paused nodes with a declared output shape (dialog: schema-validated editor, original shown read-only, reason field; absent — not disabled — when the node has no shape, per truthful-UI); `Rerun from here` on settled nodes (dialog: rerun-set preview — the node + transitive dependents — and reason field; absent while the node is parked or the run is mid-generation, US-021.EC-1/EC-2).
- **States to design**: amend dialog (edit / validation failure / conflict on concurrent amend US-004.EC-3); rerun dialog with rerun-set preview; both verbs' deterministic daemon answers rendered as information (existing `LoopControlAnswer` pattern).
- **Artboard**: `docs/design/opendesign/graph-eng/graph-eng-node-verbs.html`.

### S6. Run page · inspect sheet

- **Today**: `loop-run-inspect-sheet.tsx:35-45` renders generations, verdicts, resolved runtimes, frames.
- **Change**: each generation row gains `Compare…` (opens S7 seeded with that generation) and `Fork from here` (opens S8); a lineage block renders `forked_from` and `forks[]` with two-way links (US-020.AC-2, US-022.AC-2); route causes and strategy settlements appear in the generation detail.
- **States to design**: lineage present/absent; fork chain (fork of a fork, US-022.EC-2).
- **Artboard**: `docs/design/opendesign/graph-eng/graph-eng-inspect-lineage.html`.

### S7. Run diff view (new)

- **Change**: a full-height comparison surface launched from S6 (and deep-linkable): generation↔generation within a run, or run↔run for the same loop. Per-node rows grouped by change kind (`changed / rerun / skipped / carried / verdict`), matching the CLI diff vocabulary in `_dx.md`; unchanged summarized, large outputs summarized with sizes/hashes and a link to full content (US-019.AC-2, EC-3); input diff block for run↔run (US-020.AC-1); definition-divergence banner when the two runs pin different versions (US-020.EC-1); live-side label when one run is still executing (US-020.EC-3).
- **States to design**: empty diff (US-019.EC-1); carried-forward marking (US-019.EC-2); cross-loop rejection is CLI/API-only (the UI never offers it — pickers are same-loop scoped).
- **Artboard**: `docs/design/opendesign/graph-eng/graph-eng-run-diff.html`.

### S8. Fork dialog (new)

- **Change**: generation picker (defaulting to the inspected one) + the loop's declared-input form pre-filled with the source run's inputs (reusing the run-form machinery behind `use-loop-run-form.ts`); submit starts the fork and navigates to the new run; validation errors render exactly as the run form's (US-022.AC-3).
- **States to design**: no overrides (pure replay, US-022.EC-4); validation failure; source content unavailable (US-022.EC-1 — submit blocked with the deterministic reason).
- **Artboard**: `docs/design/opendesign/graph-eng/graph-eng-fork-dialog.html`.

### S9. Loop-runs list + attention bell (rides the herdr-parity pipeline — ADR-006)

- **Today**: loop-runs list surfaces run status. The bell is being rebuilt by the herdr-parity program (in flight): **Needs you / Finished** sections, workspace-labeled rows, badge + title fed by a daemon-computed session summary, jump = land on the source (herdr `_uiux.md` S3; `web/src/systems/os/components/attention-bell.tsx`, `use-os-attention.ts`, `attention-model.ts`). herdr's Non-Goals explicitly leave task/loop attention semantics out (herdr `_spec.md:183`) — task approval rows already sit in the bell as the precedent pattern.
- **Change**: loop requests join the redesigned bell as rows in **Needs you**, following the task-approval-row pattern: workspace label, request kind + loop name + age, jump lands on the run page's request form (S1). The badge/title count adds the loops-owned `aggregates.pending` projection (from `_dx.md`'s requests inventory) to herdr's session `needs_you` count — composition of two daemon-computed counts, never a row-page count, per-workspace error isolation preserved. The loop-runs list gains a needs-you indicator on runs with pending requests. **Concrete seam (post-merge):** this spec edits only the `web/src/systems/os` composition modules — `use-os-attention.ts` gains a loops request source, and the bell row model receives loop rows — and never touches the session-attention contracts (the `attention-summary` endpoint/consumption, session badge derivation in `attention-model.ts`, `session_attention_changed` handling, the notifier). This surface lands after herdr-parity merges (a global precondition of the spec), integrating with the merged files as they then exist.
- **States to design**: loop-request row in Needs you (pending / near-expiry); resolved/expired request never renders as a live row (it leaves the section, mirroring herdr's no-dismiss inbox-drain behavior); list-row indicator; stale/disconnected states inherit the herdr bell's existing treatment unchanged.
- **Artboard**: `docs/design/opendesign/graph-eng/graph-eng-bell-requests.html` — **iterates on `docs/design/opendesign/herdr-parity/herdr-parity-bell.html` §01 with `herdr.css` grammar** (never a parallel bell design): adds only the loop-request row kind. In bell contexts the row follows herdr's signal vocabulary (needs-you = danger); the loops-domain warning mapping applies only inside the run page.

### S10. Editor · palette + inspector

- **Today**: `loop-editor-canvas.tsx:34-53` + palette/inspector parts (`loop-editor-palette.tsx`, `loop-editor-inspector.tsx`, field parts); linter dock renders shared-linter diagnostics; `isValidConnection` is a UX hint only (the Go linter owns invariants).
- **Change**: palette gains `ask` (control) and `route` (control) — **additive**: no existing kind is removed or replaced; inspector gains — ask: prompt/context/expect/responders/expires fields (expect editor reuses the JSON-schema field pattern from `loop-editor-json-field.tsx`); route: ordered routes list (condition + target picker constrained to forward nodes) + required default picker; fan-out: strategy field (kind, threshold, `missing: acceptable` acknowledgment), `bind_as`/`index_as` names; action nodes: review block (decisions, `when`, prompt, responders, on_reject route, expires); condition-bearing nodes: `on_eval_error` override. Linter dock surfaces the new codes (`route_default_missing`, `route_target_invalid`, `strategy_coverage_undeclared`, `strategy_threshold_invalid`, `iteration_name_conflict`, `unknown_parameter` for the removed field). On the canvas, the route node card renders its ordered routes as rows — condition summary (or `default`) with a per-route source handle, so each route's edge departs from its own row (the Sim condition-block pattern, `analysis/sim-uiux.md` C13) — and a review-bearing action node exposes its `on_reject` edge the same way; palette search and quick-add ergonomics land with S12.
- **States to design**: each new inspector section (valid / lint-error); route list reordering (declaration order is the tiebreak, US-009.EC-1); DSL view round-trip of every new field (the bijective codec keeps Graph/DSL toggle lossless).
- **Artboard**: `docs/design/opendesign/graph-eng/graph-eng-editor-route-ask.html`.

### S11. Definition detail · body DAG

- **Today**: `loop-body-dag.tsx:12-15` — horizontal spine of neutral node cards, class glyph carries identity, color is state-only; `fanOutSummary` from `lib/loop-graph.ts`.
- **Change**: glyphs for `ask` and `route`; fan-out card summary includes strategy + iteration names; route card lists route count + default.
- **States to design**: read-only rendering of the new kinds; run-page DAG stays out of scope (timeline remains the run view — the live topology view is an explicit non-goal of this spec).
- **Artboard**: covered by `docs/design/opendesign/graph-eng/graph-eng-editor-route-ask.html` (detail column).

### S12. Editor · chrome & canvas ergonomics (addendum, 2026-08-16)

- **Today**: one hardcoded 3-column grid (`loop-editor.tsx:155`, duplicated in the skeleton at `:238`) — palette fixed 170/190px (`hidden lg:flex`, `loop-editor-palette.tsx:35`), inspector fixed 320/344px and always visible (`loop-editor-sidebar.tsx:69`); no chrome state persists anywhere in the loops system, and the linter-dock fold resets on remount (`loop-linter-dock.tsx:30`). The canvas mounts `Background` only: no undo/redo, no clipboard/duplicate, no context menus, no multi-select consumer, no palette search or drag source (add is click-only at rightmost+200px — `loop-editor-store-draft.ts:14-18`), default edges (no labels, no delete affordance). The toolbar already owns zoom −/%/+, fit, and auto-layout (`loop-editor-toolbar.tsx:82-98`) — ahead of Sim, which ships no zoom chrome at all; no minimap is planned (Sim has none either).
- **Change** (competitor citations: `analysis/sim-uiux.md` §2):
  - **Collapsible rails, calm defaults (C1–C3).** Both rails become collapsible with per-user persisted state (`@xstate/store/persist`, mirroring `web/src/systems/session/hooks/use-session-inspector-state.ts`; inspector width resizable via `ResizablePanel*` + `useDefaultLayout` as in `web/src/systems/network/components/shell/network-shell.tsx:86-91`). Default: both collapsed — full-bleed canvas. The palette renders one of three modes from a single `useSidebarViewport`-driven ladder replacing today's CSS-only `lg` gates — expanded rail / collapsed (the existing toolbar `LoopEditorPaletteMenu`, generalized past its `lg:hidden` rule) / drawer-menu on narrow viewports; the expanded rail gains a search field with arrow-key navigation (C9). The inspector auto-opens on node selection, palette add, and lint-dock reveal, and its Contract tab stays reachable from the collapsed toggle. Collapse toggles live in the toolbar (`PanelLeft`/`PanelRight`, the `channel-toolbar.tsx:136-148` button contract — the only in-app rail-toggle precedent), with canvas-scoped `[` / `]` as the additive keyboard path. Chrome state is one flat per-user store (`use-loop-editor-chrome-state.ts`, key `compozy:loops:editor-chrome:v1`, no per-loop key). The skeleton mirrors whichever grid state persists (both grid literals update together).
  - **Quick-add (C6–C8).** An editor quick-add dialog (existing `Command*` primitives) bound to canvas-scoped plain `a` and double-click on the pane — ⌘K belongs to the shell command palette and ⌘-chords stay shell-owned (collision table in `analysis/s12-integration.md` §1): add any palette kind at the visible viewport center, or jump-to-node (id/kind search → reveal); Escape closes with `preventDefault` so the shell overlay handler yields. Connection-drop quick-add: releasing a dragged connection over empty canvas spawns an inline picker at the drop point that creates and wires the node in one atomic draft transition. Palette items become drag sources with a card-silhouette drag preview; click-to-add moves to viewport-center placement.
  - **Clipboard + multi-select (C11).** In-memory clipboard driven by menu verbs (Duplicate · Copy · Paste — no ⌘-chords in this scope); marquee and modifier-click multi-select (already-active xyflow defaults) feeding move/delete/duplicate only — the inspector stays single-selection: the accent ring marks the inspector focus, a lighter state marks other members; arrow-key node traversal in canvas reading order.
  - **Node context menu + scoped keys (C5, C10).** A context menu on the node card (Duplicate · Copy · Paste · Rename → focuses the inspector id field · Delete; items self-censoring on read-only; `ContextMenuShortcut` hints) via the declarative `ContextMenu*` trigger wrapped around the node root — the `os-window-deck.tsx` pattern with its keyboard-parity helper; no pane menu (no in-repo precedent for the controlled anchored form — double-click on the pane opens quick-add instead). All editor bindings are canvas-scoped plain keys with the permission-dock guard (no modifiers, editable-target check, `.react-flow` containment): `a` quick-add, `[` / `]` rails.
  - **Edge ergonomics (C12–C14).** A custom edge with a generous interaction width and a ✕ delete affordance at the selected edge's midpoint; Escape cancels an in-flight connection; route edges depart from per-route rows and carry the condition pill (S10); fit view subtracts open rails so content centers in the visible canvas. Route handles and edge types are **derived display state only** — matched from the route node's `routes[]`/`default` by edge target, never written into definition edges (the daemon drops unknown edge keys — `internal/loop/dsl/graph.go:104`); deleting a node prunes its edges in the same transition (`onNodesDelete`), closing today's orphan-edge gap.
  - **Dock persistence (C15).** The linter-dock fold joins the persisted chrome state.
- **States to design**: default calm state (both rails collapsed, full-bleed canvas); palette expanded with search active vs zero-hit; inspector auto-opened on selection vs manually collapsed while a node stays selected; quick-add dialog (canvas-scoped `a` / double-click); connection-drop picker with its live edge; node context menu (editable vs read-only sets); marquee multi-select (focus ring vs member state); read-only definition (clipboard/delete verbs absent; layout, navigation, and collapse verbs stay).
- **Artboard**: `docs/design/opendesign/graph-eng/graph-eng-editor-chrome.html`.
- **Coverage note**: ergonomics addendum with no `_user_stories.md` coverage — test contract lives in `_tests.md` (E2E-031..E2E-034) and the work is assigned to task_08 (enriched 2026-08-16; subtask 8.9, VC-15/VC-16). Integration plan with the shortcut collision table, draft-store changes, and owning test suites: `analysis/s12-integration.md`. Dependency: the route-row edge treatment presupposes S10's route/ask palette + field-schema work (`route`/`ask` exist only in Go today). Two pre-existing gaps ride along as fixes: `nodeAdded` never marks `positionsDirty`, and node deletion leaves orphan edges.

## Component plan (design → production mapping)

### Rules

- Every request form derives from the daemon-persisted expected shape — no hand-authored per-loop forms; the manual-wait payload editor is the base.
- Decision buttons render only the request's persisted `decisions` set — an absent decision is absent, not disabled.
- Diff rows are data, not prose: one row per node change, grouped by change kind, matching the CLI vocabulary 1:1.
- All new mutations follow the existing invalidation sets (`runDetail`, `runsByWorkspace`, `nodeInventoryByWorkspace`) plus a new `requestsByWorkspace` key; no optimistic paints.
- Editor chrome state (rail collapse, inspector width, dock folds) persists per user via the session-inspector store pattern; pan/zoom/selection stay in the in-memory store and none of it ever enters the URL (Sim's documented carve-out, `analysis/sim-uiux.md`).

### New `@compozy/ui` primitives

None expected — forms, sheets, dialogs, badges, tables, and diff-row layout compose from the existing inventory (`packages/ui/src/index.ts`); any exception must be justified against it before a board iteration.

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
| `LoopEditorQuickAdd` (canvas-scoped `a`/double-click dialog + connection-drop picker) | `CommandDialog`/`Command*` + palette model | S12 |
| `LoopEditorEdge` (interaction width, midpoint delete, route condition pill — derived display state) | xyflow `BaseEdge` + `EdgeLabelRenderer` + chip primitives | S10, S12 |
| `LoopEditorNodeMenu` (context menu on the node card) | `ContextMenu*` + `ContextMenuShortcut`/`Kbd` | S12 |
| Editor chrome state: `use-loop-editor-chrome-state.ts` (persisted collapse/width/dock, `compozy:loops:editor-chrome:v1`) | `@xstate/store/persist` + `useDefaultLayout` + `ResizablePanel*` | S12 |

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
| Route condition edge pill | neutral chip with the condition summary; `default` renders muted |
| Selected-edge delete affordance | icon button + danger token at the edge midpoint |
```
