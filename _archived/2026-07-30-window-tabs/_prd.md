# PRD: Window Tabs (the Deck)

## Overview

CompozyOS presents the workspace as a desktop: floating and tiled windows, virtual desktops, a dock, a menubar. Today every surface — an app route, a drilled-in detail, a live agent session — demands its own window. An operator supervising several agents while keeping Tasks and Marketplace at hand ends up with a sprawl of frames competing for screen space, and the tiling system's existing "stacked" arrangement makes it worse: stacked windows have no visible switcher, so every member behind the front one is unreachable by pointer — a live defect.

Window tabs give every window the ability to multiplex surfaces, cmux-style. A thin deck above the window head holds the tab row; the head below belongs entirely to the active tab, with its full identity or breadcrumb contract. A tab is not a lesser thing than a window — it **is** a window, grouped into a shared frame (ADR-001). One concept covers floating frames and tiled panes alike, the unreachable-stack defect disappears, and the deck doubles as an attention queue: each tab shows its session's live state, so a supervisor sees which agent needs them without switching.

The feature serves three audiences: **operators** organizing many surfaces into few frames, **supervisors** running many concurrent agent sessions who need glanceable attention routing, and **agents** themselves, which must be able to inspect and arrange the workspace's tab topology through the same structured management surfaces that already govern windows and desktops.

## Goals

After this ships, users can do the following, none of which is possible today:

- Combine any windows into one tabbed frame — by drag, by menu, by keyboard, or by an agent's command — and tear any tab back out into a standalone window, losslessly.
- Keep several surfaces of the same app open at once, each with independent view state (ADR-002): watch one task run while browsing the task catalog.
- Reach every member of a tiled stacked pane by pointer, ending the unreachable-stacked-window defect.
- Read the live state of every grouped session (running, needs input, done) from the tab row without activating anything, and jump to any open tab anywhere via palette search.
- Close tabs by reflex, safely: closing never touches the underlying session, and reopening restores the tab — surface and navigation position — newest first (ADR-004).
- Restart the daemon or browser and get every group back exactly: membership, order, pins, per-tab navigation, last-active tab (ADR-003).
- Let agents list, open, group, activate, reorder, pin, and close tabs with structured output and deterministic errors, at full parity with human ability (see Agent Manageability).

The system newly guarantees: a single-tab window renders today's chrome exactly — the deck costs nothing until the second tab; and the head never crowds — route views move to the toolbar strip on every route (ADR-007), so breadcrumbs always get the head's full width.

## Success Criteria

Each criterion is binary and observable from outside the system:

1. Two windows can be grouped and ungrouped via each path — drag, context menu, palette, keyboard, agent command — with no path losing tab order or navigation state.
2. A stacked tiled pane created through the existing stack arrangement renders a deck, and every member is reachable by pointer.
3. Killing and restarting the daemon and client restores groups, order, pins, per-tab navigation stacks, and last-active tabs with zero loss.
4. Closing a tab hosting a running session leaves the session running; the dock still signals it; ⌘⇧T restores the tab at its prior navigation position.
5. Two clients on one workspace hold different active tabs in the same group simultaneously, while membership changes replicate live to both.
6. An agent can enumerate the full tab topology and perform every tab mutation through structured commands with documented, deterministic error outcomes — outcomes identical across the command surfaces.
7. Saved layouts (export, apply, profiles) round-trip tab groups faithfully.
8. No route renders peer views inside the head anymore; the amended design-system documents match the shipped product.

## User Stories

Full catalog: [Full user stories](_user_stories.md). Index by area:

- **US-001–US-004 — Grouping & deck**: forming frames, deck mount/unmount at the 2-tab boundary, active-tab ownership of head/strip/content, truthful tab identity at any width.
- **US-005–US-010 — Tab lifecycle**: ⌘T with destination picker, context-menu destination choice on launch surfaces, frictionless close, multi-level reopen, pinning, bulk close.
- **US-011–US-012 — Per-tab navigation**: independent navigation stacks, drill-in confinement, derived leaf labels.
- **US-013–US-015 — Drag & merge**: tear-out, visible cancelable drop targets, menu-based merge-all and move-to-window.
- **US-016–US-017 — Multi-instance**: explicit second instances, focus-first-cycle-on-repeat launch behavior.
- **US-018–US-019 — Attention & status**: live state on tabs, signals that outlive tabs via the dock.
- **US-020–US-021 — Keyboard & palette**: the full shortcut contract, cross-window tab search.
- **US-022–US-023 — Persistence**: restart fidelity, per-client active tab over shared membership.
- **US-024 — Tiled integration**: per-pane decks inside layouts.
- **US-025–US-028 — Agent management**: structured inspection, full-parity mutation, layout round-trip.
- **US-027 — Extensibility**: tab lifecycle events for automations.
- **US-029 — Route views amendment**: views in the strip everywhere.

## Core Features

### 1. The deck

A thin tab row (reference: 37px) above the window head, rendered only while a frame holds two or more tabs. It carries the OS window controls (which migrate up from the head while the deck exists), the tab segments, the new-tab button, and a drag region. Tabs are compact — root-app glyph or state dot plus the current leaf label — sized between a legible minimum and maximum (reference: 96–180px); past capacity the row scrolls horizontally rather than degrading below legibility. The active tab visually fuses with the head surface below, browser-style. Hover reveals a close affordance; middle-click closes; tooltips carry the full path. (US-001–US-004)

### 2. Unified window groups

Grouping is one concept everywhere (ADR-001): a floating frame with multiple tabs and a tiled pane with multiple tabs are the same thing wearing the same deck. The existing stacked arrangement in tiled layouts becomes this feature and gains the deck as its switcher (US-024). Every tab is a full window to the rest of the OS — window lists, dock, agent commands see real windows.

### 3. Per-tab head and navigation

The 44px head belongs to the active tab: root identity (glyph + title + count), breadcrumb (back chevron, clickable parents, collapse beyond two, leaf at full width), or document identity (state dot + session title + trail) render per tab with the established contracts. Each tab owns an independent navigation stack; drilling in never spawns a window or tab, and back (⌘[ or chevron) pops only the active tab's stack. Tab labels are always derived from the tab's current leaf — there is no manual rename. (US-003, US-011, US-012)

### 4. Tab lifecycle

⌘T opens a new empty tab whose content invites choosing a destination through the palette; the deck's + button does the same. Launch surfaces (dock, palette, session rows, openable list items) open windows by default — never a surprise tab — and their right-click context menu offers explicit destinations: new window, tab in the focused window, new instance (ADR-005). Close is instant and silent for any tab state; the hosted session always survives; the window closes when the last tab closes. ⌘⇧T reopens closed tabs newest-first, multiple levels deep, restoring surface and navigation position. The tab context menu carries close, close others, close to the right, pin/unpin, move to new window, and merge actions; ⌥-click on a close button closes the others. (US-005–US-010)

### 5. Pinned tabs

Pinning collates a tab at the deck's left edge, shrinks it toward glyph-only, and removes its close affordances; ⌘W refuses pinned tabs, and closing one requires unpinning first. Pins persist durably and survive regrouping. (US-009; ADR-006)

### 6. Drag, merge, and their menu twins

Dragging a tab off the deck births a standalone window at the pointer, navigation stack intact; dragging a window over another's deck shows a clear insertion affordance before anything commits, and dropping merges at that point — cancelable mid-gesture, reversible after. "Merge all windows" (current desktop) and "Move tab to new window" exist as menu and palette actions so grouping never requires pointer precision. (US-013–US-015; ADR-005, ADR-006)

### 7. Multi-instance apps

Every app may hold multiple simultaneous instances, each with independent navigation and view state (ADR-002). Default activation focuses the existing instance (cycling on repeat); creating another is always explicit. Deep links and external opens land on the most recently used instance rather than spawning duplicates. (US-016–US-017)

### 8. Attention and status signaling

Tabs surface their hosted surface's live state without activation: pulsing running dot, accent attention badge with count for needs-input, quiet idle/done forms. Accent appears only for genuine state or attention — selection stays neutral. Signals outlive tabs: the dock's per-app indicator lights while any instance exists, needs-input badges persist after a tab closes, and activating from the dock routes to the instance that needs attention first. (US-018–US-019)

### 9. Keyboard and palette

The standard tab-action contract is rebindable like existing shortcuts and discoverable in the palette: ⌘T new tab · ⌘W close tab (window when last) · ⌘⇧T reopen · ⌃Tab / ⌃⇧Tab cycle · ⌘1–9 positional jump (9 = last). The established fixed ⌘[ navigation chord goes back within the active tab. Palette search lists every open tab across windows and desktops as a distinct result group; selecting one switches desktop, focuses the window, and activates the tab in a single action. (US-020–US-021; ADR-006)

### 10. Persistence

Group membership, tab order, pinned flags, and every tab's navigation stack are durable workspace state, identical for all clients and restored exactly across restarts. Which tab is active is per-client — like focus today — with each group durably recording its last-active tab as the restore point for fresh clients and restarts (ADR-003). (US-022–US-023)

### 11. Agent manageability

Everything a human can do to tabs, an agent can do through structured surfaces: enumerate topology (membership, order, active/last-active, pins, per-tab surface identity), open-as-tab, group, ungroup, activate, reorder, pin, close — with structured output, revision-safe mutations, deterministic error outcomes, and identical behavior across the CLI, the workspace API, and the local socket surface. In-session agents get the equivalent native tools; the official Compozy skill's window-management reference teaches the new operations. Live streams announce tab changes without polling. Saved layouts and layout profiles round-trip groups. (US-025, US-026, US-028)

### 12. Extensibility

Tab lifecycle changes (open, close, group, ungroup, and durable activation — the recorded last-active commit) join the existing window-manager event family as typed, discoverable events with workspace/window/group identity in their payloads, so automations and extensions can respond to grouping the way they already respond to window moves. Per-client active-tab flips are presentation-only and deliberately not durable events; the coalesced durable activation commit is the observable record. (US-027)

### 13. Route views amendment

Peer route views (List | Kanban and equivalents) move from the window head to the toolbar strip's leading slot on every route — single-tab windows included — with strip order: views · filters · spacer · display-mode. The head is always two-element (identity/breadcrumb + trail), and the canonical design-system documents are amended in the same program so rules and product agree (ADR-007). (US-029)

## Business Rules

1. **Deck threshold.** A frame renders a deck if and only if it holds ≥2 tabs. At one tab, the deck unmounts and window controls return to the head; the single-tab window is chrome-identical to today.
2. **Tab = window.** Every tab is a full window with its own identity, surface, and navigation stack. Grouping and ungrouping never convert, copy, or truncate state.
3. **Head ownership.** The head below the deck always renders the active tab's identity contract (root / breadcrumb / document) and its trail (status + ≤2 actions). It never mixes elements from two tabs and never hosts route views.
4. **Navigation confinement.** Drill-in navigates within the active tab. It never creates a window or a tab. Back affects only the active tab's stack; back at root is a no-op.
5. **Derived labels.** A tab's label is its root-app glyph (or state dot for documents) plus its current leaf. Labels update as the tab navigates. Manual renaming does not exist.
6. **Close is non-destructive.** Closing a tab never stops, pauses, or modifies the hosted session or surface data. Close requires no confirmation in any state. The window closes when its last tab closes.
7. **Reopen contract.** Closed tabs are restorable newest-first, multiple levels deep, with surface and navigation position; restoration targets the original group when it exists, otherwise a standalone window. When a restored surface no longer resolves, the tab opens on the nearest truthful parent state with a clear notice.
8. **Pin protection.** Pinned tabs collate left, persist durably, refuse ⌘W/hover-close/middle-click, and reorder only within the pinned region. Unpinning restores normal behavior at the region boundary.
9. **Durability split.** Group membership, tab order, pins, and per-tab navigation stacks are durable, workspace-scoped, and identical for all clients. Active tab is per-client. Each group durably records last-active as the restore seed; the recorded value affects only future restores, never other live clients.
10. **Default destination.** Normal activation of a launch surface opens or focuses a window/instance — a tab is never created implicitly. Tab creation happens only through explicit gestures: ⌘T/+, context-menu choice, drag-merge, menu merge verbs, or agent command.
11. **Focus-first, explicit duplication.** With ≥1 instance of an app open, plain activation focuses (and cycles on repeat); new instances arise only from an explicit "new instance" action or an agent command. Deep links land on the most recently used instance.
12. **Grouping is total.** Any windows can group, regardless of app or sameness, across the deck threshold, in floating and tiled containers alike. Merge scope for "merge all" is the current desktop.
13. **Drag safety.** A merge commits only on release over a visible insertion affordance; anything else cancels. Escape cancels a tab drag. Every grouping operation has a lossless inverse.
14. **Signal discipline.** Accent color in the deck marks state or attention only (running, needs-input, badge counts). Selection and hover are neutral. No decorative accent.
15. **Attention outlives tabs.** Needs-input and running signals continue on the dock and sessions surfaces after a tab closes; answering a signal anywhere clears it everywhere live.
16. **Agent parity.** Every tab read and mutation available to humans exists as a structured command with identical behavior and payloads across the management surfaces, revision-checked like existing window commands, with documented deterministic errors for: unknown window, unknown group, non-member activation, stale revision, malformed group spec.
17. **Event truthfulness.** Durable tab changes emit typed events tied to one revision change per user-level operation (bulk operations arrive as one coherent change). Per-client activation emits no durable event.
18. **Layout round-trip.** Exported layouts, applied layouts, and layout profiles carry groups completely (membership, order, pins, last-active). Pre-tabs artifacts are not converted: saved layout documents from before this feature are rejected with a clear version error (re-exporting recreates them), and persisted window arrangement from before this feature is discarded once at load, reinitializing a fresh layout — sessions, tasks, and all runtime data untouched.
19. **Route views placement.** Peer views render in the strip's leading group on every route that has them; the head never hosts them. Strip order is views · filters · spacer · display-mode.
20. **Keyboard integrity.** Tab-action shortcuts follow the established rebinding, conflict-detection, and editable-target rules. Defaults: ⌘T, ⌘W, ⌘⇧T, ⌃Tab/⌃⇧Tab, ⌘1–9 (9 = last). The established fixed ⌘[ navigation chord pops the active tab. Single-tab degradation: ⌘W closes the window; cycling and jumps are no-ops; ⌘T mounts the deck.

## User Experience

**Personas** — see the catalog for full definitions: Operator (organizes many surfaces), Supervisor (routes attention across concurrent agents), Agent (arranges the workspace programmatically), Automation author (reacts to lifecycle events).

**First contact.** Nothing changes until the second tab: every existing window keeps today's chrome (the deck's cost-nothing rule is the onboarding). Discovery happens through the gestures users already try: dragging one window onto another reveals the insertion affordance; right-clicking a dock app reveals destination choices; ⌘T on a focused window mounts the deck with an empty tab inviting a destination via the palette.

**Primary flow — supervising agents.** A supervisor merges three session windows into one frame ("Merge all windows" or drag). The deck shows three tabs: one pulsing (running), one badged (needs input), one quiet. They click the badged tab — the head swaps to that session's document identity, the composer is exactly as left. They answer, ⌃Tab back to the runner. Closing a finished session's tab is a reflex ⌘W; the session archive remains reachable from the sessions list. A week later the same frame is still there after restarts, pins intact.

**Primary flow — two views of one app.** An operator right-clicks Tasks in the dock → "Open new instance" → "Open as tab in focused window". Two Tasks tabs now live side by side: one at List filtered to running, one drilled into a specific run. The deck labels read "Tasks" and "Run #47" — derived leaves, no rename needed.

**Primary flow — agent-arranged workspace.** An operator asks their agent to "set up my triage screen". The agent issues structured commands: open Tasks, open two sessions as tabs grouped with it, activate the needs-input one, pin the dashboard. The operator watches the frame assemble live; undo history and revision checks keep concurrent human gestures safe.

**Accessibility.** The full contract is keyboard-reachable (creation, destination choice, cycling, jumps, close, reopen, merge/move via palette and menus — drag is never the only path). Tabs expose proper list/selection semantics to assistive tech; state is never color-only (dot shape/motion + badge counts + tooltips); tooltips carry full titles for truncated labels; hit targets follow the established shell minimums; the editable-target guard keeps shortcuts out of text inputs.

**Discoverability.** Tab actions appear in the ⌘K palette with their shortcuts; the shortcuts settings surface lists the rebindable tab actions; context menus name their keyboard equivalents; the official skill reference teaches agents the same operations. The fixed ⌘[ navigation chord remains documented with active-tab back behavior.

## High-Level Technical Constraints

- **One authority.** Tab topology is part of the same daemon-authoritative, workspace-scoped window-manager state as windows and desktops today: revision-checked mutations, live replication to all clients, identical truth on every surface. No client-private tab state beyond the established per-client presentation set (active tab, alongside focus and active desktop).
- **Surface parity.** Every new read/mutation ships simultaneously on the CLI, the workspace API, the local socket surface, native tools for in-session agents, the live event stream, and the palette — with structured output and the documented error contract. A capability reachable only through the web UI is out of contract.
- **Workspace isolation.** Tab topology is workspace-scoped presentation data like the rest of window-manager state; it carries no session content of its own and must not leak across workspaces on any list, stream, or cache path.
- **Truthful UI.** Tab state renders only what the runtime models: session dots/badges bind to real session states; no invented indicators. Where the design reference conflicts with runtime truth, runtime wins.
- **Design-system conformance.** The deck implements the chosen reference contract (dimensions, palette discipline, motion) from the canonical token source and amended shell rules — no invented values; the reference file, shell rules, and migration map are updated with the product.
- **Performance.** Tab switching feels instantaneous (surface state already resident — no reload on switch); restore of a typical workspace is not perceptibly slower than today's window restore; decks stay responsive at hundreds of tabs (scroll + palette as the guaranteed escape), validated by volume tests at the projection level while interactive end-to-end coverage stays at a bounded envelope.
- **Greenfield hard cuts.** The tab model replaces the UI-less stacked arrangement's presentation and replaces persisted window-manager state in one move: no dual formats, no compat shims, no converters — pre-tabs saved documents are rejected with a deterministic version error and pre-tabs persisted arrangement is discarded once at load. Obsolete "stacked = hidden members" behavior is deleted, not preserved.

## Non-Goals (Out of Scope)

- **Hover peek / inline reply without switching** — deferred by user decision to a future iteration (ADR-006). The deck routes attention; answering happens in the tab.
- **Live sub-labels on tabs** (last-output lines, ports, branch info) — does not fit legible 96–180px segments; tooltips and the head carry detail (ADR-006).
- **Ephemeral/preview tabs** — unnecessary by design: drill-in never spawns tabs (Business Rule 4), so the tab-explosion problem preview tabs solve cannot occur here.
- **A global "prefer tabs" preference** — rejected in favor of one predictable default plus explicit gestures (ADR-005); revisitable post-v1 only on demonstrated demand.
- **Manual tab renaming** — labels are derived truth (Business Rule 5); renaming would let labels drift from what a tab actually shows.
- **Tab sharing/sync across workspaces** — tabs are workspace-scoped presentation; cross-workspace composition is a different feature.
- **New attention semantics** — the deck surfaces existing session states; it invents no new lifecycle states, counters, or notification channels beyond the tab events named here.

## Architecture Decision Records

- [ADR-001: Window tabs are grouped windows (unified tab model)](adrs/adr-001.md) — a tab is a full OS window in a shared frame; one concept for tiled and floating; absorbs the UI-less stack.
- [ADR-002: Apps become multi-instance](adrs/adr-002.md) — any app may hold N simultaneous instances; focus-first launch, explicit duplication, aggregate dock signaling.
- [ADR-003: Tab persistence split](adrs/adr-003.md) — membership/order/pins/navigation durable per workspace; active tab per client with durable last-active as restore seed.
- [ADR-004: Frictionless tab close with multi-level reopen](adrs/adr-004.md) — no confirmation ever; sessions survive; ⌘⇧T restores newest-first with navigation intact.
- [ADR-005: Windows by default; tabs are explicit, context-menu-first](adrs/adr-005.md) — no global preference; right-click destination chooser on launch surfaces; drag with visible cancelable affordances.
- [ADR-006: v1 extras](adrs/adr-006.md) — ⌘K tab search, menu-based merge/move, pinned tabs in; peek and live sub-labels deferred.
- [ADR-007: Route views move to the strip everywhere; deck sanctioned as third bar](adrs/adr-007.md) — global amendment at ship; head stays two-element; DS documents updated with the product.

## Open Questions

- **Small-screen/touch behavior.** The chosen reference contract is desktop-grade (hover, drag, keyboard). Below the shell's small-screen breakpoint the OS presentation already changes shape; whether decks render there (e.g., switch-only without drag) or windows stay untabbed on small screens needs a product decision before the small-screen surface is touched. No behavior in this PRD depends on the answer.
