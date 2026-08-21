# UI/UX Change Map: Agent Comms — Typed Calls, Mailbox, and Subagents

Every UI surface this feature touches: where it lives today, what changes, which
states must be designed, and the reference artboard each surface needs. Artboards
land under `docs/design/opendesign/agent-comms/` and become the visual contracts
the implementation tasks cite.

Companions: `_spec.md` Part I (behavior authority), `_user_stories.md` (states
come from ACs/ECs), `_dx.md` (the non-UI half of the surface).

## Design constraints (apply to every artboard)

- **Signal palette is information, never decoration.** Proposed semantic mapping
  (final call belongs to the design pass): call `running` → neutral; `completed` →
  `#5FBF85` success; `invalid-result` / `failed` / `expired` → `#E0635A` danger;
  `canceled` / `timeout` → `#D6A647` warning; needs-you rows reuse the locked badge grammar
  (`web/src/systems/session/lib/session-badge.ts:37` + herdr-parity
  DESIGN-NOTES) — one tone per attention class, identity by glyph, color for
  state only (the `LoopBodyDag` rule).
- **Truthful UI.** No "mark read" control (nothing daemon-backed models it —
  same rule the bell follows, `attention-bell.tsx:44-53`); counts come from a
  daemon summary projection, never from loaded pages
  (`web/src/systems/os/lib/attention-model.ts:1-17`); stale sources contribute
  zero but keep rows clickable; `verdict: extracted` renders as extracted, never
  as returned; cost renders only through `describeCost()`
  (`web/src/lib/cost-provenance.ts:94`).
- **Flat depth; tokens only** (`packages/ui/src/tokens.css` + `DESIGN.md`);
  every uppercase label is `Eyebrow`; no helper prose under headings.
- **Keyboard**: tree fully navigable (↑↓ traverse, ←→ collapse/expand, Enter
  opens detail); the Agents window honors the shell's window/focus keys.
- **Copy sources**: `COPY.md` register; state names come from the `_dx.md`
  vocabulary (`running`, `completed`, `invalid-result`, `parked`…) — never
  invented synonyms.

## Surface map

The **Agents app already exists** (`web/src/systems/os/lib/app-catalog.ts:70` —
id `agents`, title "Agents", dock group 2, `paths: ["/agents"]`) with three
locations driven by the window controller
(`web/src/systems/os/apps/agents/agents-window.tsx:10` +
`agent-window-location.ts`): catalog (`/agents`), agent detail
(`/agents/$name`), agent settings (`/agents/$name/settings`); session windows
live under `/agents/$name/sessions/$id`. This feature **extends that app** with
new locations — it registers no new app.

| #   | Surface                                     | Kind             | Core change                                        | Stories                        |
| --- | ------------------------------------------- | ---------------- | -------------------------------------------------- | ------------------------------ |
| S1  | Agents app — Activity location              | new location     | Delegation trees across the workspace, live        | US-028, US-020, US-023, US-031 |
| S2  | Agents app — call detail location           | new location     | Call drill-in: contract, timeline, typed result    | US-028, US-007..011, US-005    |
| S3  | Agents app — inbox location                 | new location     | Mailbox view with delivery outcomes                | US-028, US-012..016            |
| S4  | Agents app — catalog + agent detail         | modify           | Descriptions, live instance counts, calls per agent | US-021, US-022, US-024         |
| S5  | Session detail — Calls panel                | modify           | New inspector tab + timeline call/result rows      | US-029, US-017                 |
| S6  | Attention surfaces                          | modify           | Calls badge + bell rows for call needs-you states  | US-030                         |
| S7  | Dock                                        | modify           | `calls` badge on the existing Agents app           | US-028, US-030                 |

### S1. Agents app — Activity location

- **Today**: the app opens on the catalog location
  (`web/src/systems/os/apps/agents/agents-catalog-location.tsx`); no location
  renders live delegation activity. The closest existing precedent for the tree
  itself is the session thread list
  (`web/src/systems/session/lib/session-hierarchy.ts:50` — cycle-guarded
  parent→child tree; renderer `session-list-thread.tsx:33`).
- **Change**: a new **Activity** location (`/agents/activity`, reachable as a
  lane from the catalog) — every delegation tree in the workspace, live: caller
  → children → grandchildren (depth ≤ 3), one row per call with state, agent
  identity (owner-palette color + glyph), age, and trailing cost/result stats.
  Trees collapse; urgency escalates to the collapsed toggle (existing
  `childSessionSignalTone` rule). Selecting a row opens S2. There is no
  dedicated Revive control anywhere: revival IS calling or messaging a parked
  child (US-018), so the affordances are call-again and message — never an
  invented button. The window
  controller (`agents-window.tsx:10`) and `agent-window-location.ts` gain the
  new location kinds; live work gates on `useWindowLiveDataEnabled`.
- **States to design**: default with 2-3 live trees (US-028.AC-1); queued /
  running / completed / invalid-result / completed-without-result / failed /
  canceled / timeout / expired rows
  (US-001, US-003.AC-4, US-005, US-008, US-009, US-019); parked children
  distinct from running and gone (US-018.AC-3); parent-drained subtree (US-020
  — the "stop subtree" control invokes session stop with `subtree`, exactly the
  `_dx.md` operation); deep/wide tree with collapse + focus (US-028.EC-2);
  empty state teaching the feature (US-028.EC-1); SSE-stale (US-028.EC-3);
  stale-action feedback (US-028.EC-4).
- **Artboard**: `agent-comms-activity-tree.html`.

### S2. Agents app — call detail location

- **Today**: nothing renders a call; the app's detail location renders agent
  definitions (`web/src/systems/os/apps/agents/agent-detail-location.tsx`).
  Existing grammars to reuse: tool-call rows
  (`web/src/systems/session/components/tool-call-card.tsx:14`), run cards
  (`RunCard`), JSON rendering (`JsonViewer`), cost (`describeCost()`). New
  location: `/agents/calls/$callId`.
- **Change**: drill-in for one call: header (agent, caller, child session link,
  state, depth, idle-TTL state — suspended while running, counting while
  parked), the prompt, the result contract (digest + collapsed
  schema view), state timeline (created event → queued → running → settled, with repair
  attempt and extraction provenance when present), the typed result
  (schema-aware record view with bounded preview + "open full payload"), cost
  line, and the real controls: cancel (in-flight), call again (terminal),
  message child — every control maps 1:1 to `_dx.md` operations.
- **States to design**: running (cancel available); completed with `returned` /
  `extracted` / `repaired` verdicts (US-007, US-011, US-008); invalid-result
  with both attempts' errors verbatim (US-008.AC-2); completed-without-result
  (US-009); canceled with superseded late result (US-005.AC-2); timeout with
  the opt-in deadline shown in the header when set (US-005.AC-3, deadline_at +
  remaining while running); over-budget overflow (US-010); result too large —
  preview + fetch (US-017.AC-3); untrusted text (message bodies, descriptions)
  rendered inert (US-021.EC-2, US-016.AC-1).
- **Artboard**: `agent-comms-call-detail.html`.

### S3. Agents app — inbox location

- **Today**: no message surface exists. The bell's two-section grammar
  (`attention-bell.tsx:55`) and dock rows (`session-decision-dock.tsx`) are the
  closest patterns. New location: `/agents/inbox` (workspace-wide, filterable
  per agent).
- **Change**: per-agent mailbox: messages with direction, provenance
  (agent/operator origin stamped), delivery outcome (`delivered-into-turn` /
  `woke` / `queued` / `failed`), and age. No read/seen state exists anywhere —
  the runtime does not model it, so no unread mark renders (truthful UI).
  Compose box to message any listed agent (operator → agent path, US-013),
  disabled affordance absent (not grayed) when the target is gone.
- **States to design**: mixed inbox (US-012, US-014); queued-then-delivered
  transition (US-014.AC-2); rate-limited / dedup-dropped receipts visible
  (US-015.AC-1..2); blocked-target compose error (US-013.EC-1); empty inbox;
  cap-pressure state (US-015.AC-3).
- **Artboard**: `agent-comms-inbox.html`.

### S4. Agents app — catalog + agent detail (the roster)

- **Today**: the catalog location lists definitions
  (`web/src/systems/os/apps/agents/agents-catalog-location.tsx` +
  `use-agents-catalog.ts`) and the detail location shows one definition
  (`agent-detail-location.tsx` + `use-agent-detail.ts`); no descriptions exist
  anywhere (`internal/api/contract/agent_observe_payloads.go:28` — field added
  by this feature).
- **Change**: catalog rows gain description, scope badge (workspace / global /
  shadowed), and live instance count (running/parked children per definition).
  Agent detail gains a recent-calls list for that definition and a "Call"
  action opening a prefilled compose (prompt + optional contract) — the
  operator path to US-001.
- **States to design**: roster with shadowing (US-021.AC-3, US-024.EC-1); empty
  description rows (US-021.AC-2); zero-definitions empty state (US-022.EC-2);
  large roster with bounded rendering (US-022.EC-1); the **Call compose** flow
  (operator path to US-001): editing (prompt + optional contract), contract
  invalid (`call_expect_invalid` inline), unknown/expired target (US-001.EC-1,
  US-019.AC-1), submitting, accepted (link to the new call in S2).
- **Artboard**: `agent-comms-roster.html`.

### S5. Session detail — Calls panel

- **Today**: inspector tabs are `usage | memory | files | vault`
  (`web/src/systems/session/components/session-inspector.tsx:29-40`); timeline
  rows render markers via `session-timeline-render.tsx:275-293`; wake causes
  render in `runtime-activity-notice.tsx:29`.
- **Change**: (a) new inspector tab **Calls** — this session's calls, both
  directions (made / received), state + result preview, linking into S2; (b)
  timeline: call-created and result-arrived rows using the Marker/MarkerMeta
  grammar; the completion wake row shows "why this session woke" with the call
  identity and preview (US-029.AC-2); (c) the child side shows its bound call
  context.
- **States to design**: tab with mixed-state calls (US-029.AC-1); wake row with
  result preview (US-017.AC-1); hundreds of calls — pagination + truthful counts
  (US-029.EC-1); pruned-counterpart degradation (US-029.EC-2).
- **Artboard**: `agent-comms-session-calls-panel.html`.

### S6. Attention surfaces

- **Today**: `OsAttentionBadges {sessions?, tasks?, loops?}`
  (`web/src/systems/os/lib/attention-model.ts:32`); bell two sections
  (needs-you / finished), counts daemon-projected only.
- **Change**: widen the badge union with `calls`; needs-you rows for:
  invalid-result, completed-without-result, child blocked on a decision;
  finished rows for completed calls awaiting a look (US-017 visibility, under
  the shell's existing finished-section grammar). (No budget-exhausted state
  exists — completions are never admission-denied, ADR-011.) Signal storm
  coalesces per tree (US-030.EC-2). No new dismissal mechanics.
- **States to design**: bell with call rows among session/task rows
  (US-030.AC-1); coalesced tree row (US-030.EC-2); auto-resolved clearing
  (US-030.EC-1).
- **Artboard**: `agent-comms-attention.html` (bell + toast states only).

### S7. Dock

- **Today**: the Agents app already sits in dock group 2
  (`app-catalog.ts:70`) with no badge; badges render via `dockBadgeFor`
  (`dock-badges.ts:4`) from the `badge?: "sessions" | "tasks"` union
  (`app-catalog.ts:33`).
- **Change**: widen the badge union with `"calls"` and set it on the existing
  Agents descriptor — needs-you call states light the badge the same way
  sessions/tasks do. No new glyph, no catalog entry.
- **States to design**: covered inside S1/S6 artboards (dock strip with badge);
  no dedicated artboard.

## Component plan (design → production mapping)

### Rules

- Compose from `@compozy/ui`; artboard CSS is a visual contract, never a
  stylesheet to import. Domain components carry the `Agent` prefix
  (`compozy-ui-reuse/no-shadow-ui-primitive` is a blocking error).
- One projection module (`lib/agent-comms-tree.ts`, modeled on
  `session-hierarchy.ts`) owns tree building + cycle guard; components stay
  render-only. Query keys/options follow the `networkKeys` factory pattern.
- Counts and cost: daemon summary projections + `describeCost()` only.

### New `@compozy/ui` primitives

- **None planned.** The tree uses the existing `Tree/TreeItem/TreeItemLabel`
  exports (first real consumer — budget for its story/test/a11y pass in
  `packages/ui`, no API change expected). Result rendering starts on
  `JsonViewer` + `MetadataList`/`PropertyRow`; if a schema-aware record view
  proves reusable beyond this feature it graduates to `packages/ui` later —
  not in this change.

### New domain components

Roster changes extend the existing `web/src/systems/agent/` system and the
app's location files; call/mailbox composites land in a new
`web/src/systems/agent-comms/` system (calls span sessions and agents — they
are their own domain).

| Component | Composed from | Used by |
| --- | --- | --- |
| `AgentCallTreeRow` | `TreeItemLabel`, `StatusDot`, `Pill`, `OwnerAvatar`, `Time` | S1 |
| `AgentCallDetail` | `Panel`, `Eyebrow`, `MetadataList`, `Timeline`+`TimelineEvent`, `JsonViewer`, `CodeBlock`, `Metric` | S2 |
| `AgentCallResultView` | `JsonViewer`, `PropertyRow`, `CopyIconButton`, `DataSurface*` | S2, S5 |
| `AgentInboxRow` | `ListingRow*`, `PillDot`, `OwnerAvatar`, `Time` | S3 |
| `AgentComposeMessage` | `SearchInput`/`CommandSelect`, `Panel` | S3, S4 |
| `AgentCallCompose` | `Panel`, `CommandSelect`, `CodeBlock` (contract), `ActionResultBanner` | S4 |
| `AgentRosterRow` | `ListingRow*`, `KindChip`, `MonoId`, `Pill` | S4 |
| `AgentCallsInspectorPanel` | `DetailInspectorTab` section parts (`session-inspector-sections.tsx` pattern) | S5 |
| `AgentCallMarkerRow` | `Marker`, `MarkerMeta` | S5 timeline |

### Signal & state mapping

| Design state | Primitive + token |
| --- | --- |
| `running` | `StatusDot` neutral + `TypingDots` on active row |
| `completed` (`returned`/`repaired`) | `Pill` tone success |
| `completed` (`extracted`) | `Pill` tone success + `KindChip` "extracted" (provenance, not decoration) |
| `invalid-result` / `failed` / `expired` | `Pill` tone danger |
| `canceled` / `timeout` | `Pill` tone warning |
| `parked` | `Pill` tone neutral + moon glyph (identity by glyph) |
| needs-you rows | existing `sessionBadgeSignal` grammar — no second dictionary |
| cost | `Metric` + `describeCost()` output verbatim |
