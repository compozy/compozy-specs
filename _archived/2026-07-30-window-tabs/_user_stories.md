# User Stories: Window Tabs

Canonical behavior catalog for window tabs (the deck). Companion to `_prd.md`; consumed by
`_techspec.md` (component mapping) and `_tests.md` (coverage matrix).

## Personas

- **Operator** — a human running CompozyOS day to day: several apps and agent sessions open in one workspace, often across two browsers/monitors, keyboard-heavy. Needs to group related work into one frame, switch instantly, and trust that the arrangement survives restarts.
- **Supervisor** — the same human in multi-agent mode: many concurrent sessions, attention scattered. Treats the deck as an attention queue — which agent needs me now — and relies on state signals that are visible without activating anything.
- **Agent** — an automated manager (a Compozy session or an external program) that operates the shell through structured command surfaces. Needs to inspect, create, move, activate, and close tabs with structured output, deterministic errors, and the same power a human has.
- **Automation author** — builds automations and extensions that react to shell changes. Needs tab lifecycle signals on the same event surfaces that already announce window and desktop changes.

## Story Index

| ID     | Feature Area          | Persona           | Story                                                             |
| ------ | --------------------- | ----------------- | ----------------------------------------------------------------- |
| US-001 | Grouping & deck       | Operator          | Group two windows into one tabbed frame                           |
| US-002 | Grouping & deck       | Operator          | Deck appears at the second tab and vanishes at the last           |
| US-003 | Grouping & deck       | Operator          | Switch tabs; head, strip, and content follow the active tab       |
| US-004 | Grouping & deck       | Operator          | Tabs identify themselves truthfully at any width                  |
| US-005 | Tab lifecycle         | Operator          | Create a new tab with ⌘T and pick its destination                 |
| US-006 | Tab lifecycle         | Operator          | Right-click any openable item and choose where it opens           |
| US-007 | Tab lifecycle         | Operator          | Close a tab without fear                                          |
| US-008 | Tab lifecycle         | Operator          | Reopen closed tabs, newest first                                  |
| US-009 | Tab lifecycle         | Operator          | Pin a long-lived tab                                              |
| US-010 | Tab lifecycle         | Operator          | Bulk-close via the tab context menu                               |
| US-011 | Per-tab navigation    | Operator          | Drill in without spawning windows or tabs                         |
| US-012 | Per-tab navigation    | Operator          | Tab labels follow where each tab is                               |
| US-013 | Drag & merge          | Operator          | Drag a tab out into its own window                                |
| US-014 | Drag & merge          | Operator          | Drag a window onto a deck to merge — visibly and cancelably       |
| US-015 | Drag & merge          | Operator          | Merge and move without dragging                                   |
| US-016 | Multi-instance        | Operator          | Open a second instance of an app on purpose                       |
| US-017 | Multi-instance        | Operator          | Launch surfaces focus first, cycle on repeat                      |
| US-018 | Attention & status    | Supervisor        | Read every agent's state from the deck without switching          |
| US-019 | Attention & status    | Supervisor        | Never lose a needs-input signal by closing a tab                  |
| US-020 | Keyboard & palette    | Operator          | Navigate tabs entirely from the keyboard                          |
| US-021 | Keyboard & palette    | Supervisor        | Jump to any tab anywhere from the palette                         |
| US-022 | Persistence           | Operator          | Everything is back after a restart                                |
| US-023 | Persistence           | Operator          | Two screens, two active tabs, one shared truth                    |
| US-024 | Tiled integration     | Operator          | Tabbed panes inside tiled layouts                                 |
| US-025 | Agent management      | Agent             | Inspect groups and tabs with structured output                    |
| US-026 | Agent management      | Agent             | Manage tabs with the same power a human has                       |
| US-027 | Extensibility         | Automation author | React to tab lifecycle events                                     |
| US-028 | Agent management      | Agent             | Saved layouts round-trip tab groups                               |
| US-029 | Route views amendment | Operator          | Route views live in the strip everywhere                          |

## Grouping & deck

### US-001: Group two windows into one tabbed frame

**As an** Operator, **I want** to combine two windows into a single frame with a tab row, **so that** related work shares one screen slot instead of overlapping.

Acceptance criteria:

- AC-1: Given two floating windows, when I drop one onto the other's deck area (or use an explicit merge action), then one frame remains, showing a thin deck above the head with both tabs, the dropped window's tab active.
- AC-2: Given a tabbed frame, when I look at its chrome, then the deck shows the OS window controls, one tab per member, a new-tab button, and a drag region — and the 44px head below belongs entirely to the active tab.
- AC-3: Given any two windows regardless of app (two instances of the same app included), when I group them, then grouping succeeds — there is no "cannot merge these apps" case.

Edge cases:

- EC-1: Grouping a window that itself already has tabs → all its tabs join the target deck, preserving their order and per-tab navigation, and no tab is lost.
- EC-2: Grouping across different desktops via an explicit action → the moved window's tab lands on the target's desktop, and the origin desktop no longer shows it.
- EC-3: Two clients group different windows into the same frame at the same moment → both operations land or one is retried transparently; no tab disappears and no duplicate frame appears.
- EC-4: Attempting to group a window with itself (drag onto its own deck) → reorders within the deck instead of creating anything new.

### US-002: Deck appears at the second tab and vanishes at the last

**As an** Operator, **I want** single windows to look exactly as they do today, **so that** tabs cost me nothing until I actually use them.

Acceptance criteria:

- AC-1: Given a window with one tab, when I view it, then no deck renders and the OS window controls sit in the head — today's chrome, pixel-for-pixel.
- AC-2: Given a single-tab window, when a second tab arrives (⌘T, merge, or agent command), then the deck mounts and the controls move from the head into the deck.
- AC-3: Given a two-tab frame, when I close one tab, then the deck unmounts and the remaining tab renders as a normal single window with controls back in the head.

Edge cases:

- EC-1: Rapidly adding then removing a second tab → chrome transitions cleanly both ways with no orphaned controls or duplicated control clusters.
- EC-2: A pinned tab as the sole survivor → deck still unmounts; the pin state is retained invisibly and reappears if the window gains tabs again.

### US-003: Switch tabs; head, strip, and content follow the active tab

**As an** Operator, **I want** the whole window below the deck to belong to whichever tab is active, **so that** each surface keeps its full identity, actions, and tools.

Acceptance criteria:

- AC-1: Given a frame with a root-view tab, a drilled-in tab, and a session tab, when I click each tab, then the head swaps between root identity (glyph + title + count), full breadcrumb (back chevron, clickable parents, "…" collapse, leaf), and document identity (state dot + session title + agent/model trail) respectively.
- AC-2: Given a tab with a toolbar strip (views, filters, display modes), when that tab is active, then its strip renders below the head; tabs without a strip render none.
- AC-3: Given a tab switch, when the new tab's content appears, then it is exactly where I left it — scroll position, selection, and in-progress composer text preserved.
- AC-4: Given a click on an inactive tab's close button, when I press it, then the tab closes without first becoming active.

Edge cases:

- EC-1: Switching to a tab whose surface is still loading → the tab activates immediately and the content area shows its loading state; the deck never blocks on content readiness.
- EC-2: Switching tabs while a drag/resize gesture is in progress elsewhere in the frame → the gesture cancels or completes safely; no half-applied layout.
- EC-3: Activating a tab via keyboard while the pointer hovers another tab → keyboard wins; hover styling never changes activation.

### US-004: Tabs identify themselves truthfully at any width

**As an** Operator, **I want** every tab to stay recognizable as the deck fills, **so that** I never face a row of identical mystery labels.

Acceptance criteria:

- AC-1: Given a route tab, when I read it, then it shows the root app's glyph plus the current leaf name; given a session tab, it shows the session's state dot plus the session title.
- AC-2: Given many tabs, when segments compress, then they shrink only to a legible minimum (per the reference: 96px) and the row then scrolls horizontally; labels truncate with ellipsis, never overlap.
- AC-3: Given a truncated tab, when I hover it, then a tooltip shows the full path/title (e.g., "Tasks / Nightly triage / Run #418").
- AC-4: Given two tabs that would read identically (two "Tasks" instances at the root), when both are visible, then they are disambiguated by their distinct leaf/view state rather than bare app names.

Edge cases:

- EC-1: A tab title with hostile/very long content (a 300-character session name, emoji, RTL text) → renders safely, truncates by width, full value in the tooltip.
- EC-2: A tab whose surface has no resolvable title yet (brand-new empty tab) → shows a neutral "New tab" label until a destination is chosen.
- EC-3: 30+ tabs in one deck (100× probe) → the row scrolls; every tab remains reachable by scroll, keyboard cycling, and palette search; nothing is silently dropped.

## Tab lifecycle

### US-005: Create a new tab with ⌘T and pick its destination

**As an** Operator, **I want** ⌘T to open a fresh tab with a destination picker, **so that** adding a surface to my current frame is one gesture.

Acceptance criteria:

- AC-1: Given a focused window, when I press ⌘T (or click the deck's + button), then a new active tab appears showing an empty state that invites picking a destination via the palette.
- AC-2: Given the empty tab's picker, when I choose an app, route, or session, then the tab becomes that surface with a fresh navigation stack.
- AC-3: Given a single-tab window, when I press ⌘T, then the deck mounts (per US-002) and the new empty tab is active.

Edge cases:

- EC-1: Pressing ⌘T repeatedly → multiple empty tabs are allowed; each closes with one ⌘W; abandoning them costs nothing.
- EC-2: Dismissing the picker without choosing → the empty tab remains, labeled "New tab", closable normally.
- EC-3: ⌘T while no window is focused → creates a new single window with the picker instead (no dead gesture).

### US-006: Right-click any openable item and choose where it opens

**As an** Operator, **I want** a context menu on launch surfaces (dock apps, palette results, session rows, list items that open windows), **so that** I decide per gesture whether something opens as a window, a tab here, or a new instance.

Acceptance criteria:

- AC-1: Given any openable item, when I left-click/activate it normally, then today's default behavior happens: focus the existing window/instance or open a new window — never a surprise tab.
- AC-2: Given the same item, when I right-click it, then a context menu offers at least: "Open in new window", "Open as tab in focused window", and — when an instance already exists — "Open new instance" (per US-016).
- AC-3: Given "Open as tab in focused window" with no focused window, then the option is disabled or falls back to opening a window, stated in the menu — never a silent no-op.

Edge cases:

- EC-1: Right-click on an item that is already open as a tab somewhere → menu additionally offers "Go to tab" (focus + activate).
- EC-2: Context menu invoked on a launch surface while a modal/overlay owns the screen → menu still works or the surface is inert; no half-open menus behind overlays.
- EC-3: Keyboard-only users → the same destination choices are reachable without a pointer (menu key or palette actions).

### US-007: Close a tab without fear

**As an** Operator, **I want** tab close to be instant and non-destructive, **so that** I keep my deck clean without ceremony.

Acceptance criteria:

- AC-1: Given an active tab, when I press ⌘W, then the tab closes immediately — no confirmation — and the neighbor (per close policy) becomes active; when it was the last tab, the window closes instead.
- AC-2: Given a tab hosting a running or needs-input session, when I close it, then the session keeps running untouched in the daemon and remains reachable from the sessions list and dock.
- AC-3: Given any tab, when I hover it, then a close affordance (×) appears; middle-click also closes; a pinned tab shows no × and refuses ⌘W (per US-009).
- AC-4: Given a close, when I look at the deck, then no dialog, toast, or delay interrupted the action.

Edge cases:

- EC-1: Closing the last tab of the workspace's only window → the window closes; the desktop is empty; no error.
- EC-2: Two clients close the same tab simultaneously → the tab closes once; the second action is a harmless no-op.
- EC-3: An agent closes the tab I am looking at → my window updates live: next tab activates, or the window closes if it was the last.
- EC-4: Closing a tab mid-drag (its own drag or another tab's) → the drag cancels cleanly; the deck settles with no ghost tab.

### US-008: Reopen closed tabs, newest first

**As an** Operator, **I want** ⌘⇧T to bring back what I just closed, **so that** closing is always reversible.

Acceptance criteria:

- AC-1: Given I closed a tab, when I press ⌘⇧T, then that tab returns with its surface and navigation stack intact, into its original group when it still exists, otherwise as a standalone window.
- AC-2: Given several closes, when I press ⌘⇧T repeatedly, then tabs return in reverse close order, multiple levels deep.
- AC-3: Given a reopened session tab, when it returns, then it reattaches to the live session as it is now (not a stale snapshot).

Edge cases:

- EC-1: ⌘⇧T with nothing to reopen → quiet no-op, no error surface.
- EC-2: Reopening a tab whose session was meanwhile deleted → the tab returns to the nearest truthful state (e.g., the sessions list) with a clear notice instead of a dead document.
- EC-3: Reopening after the original group was itself closed → the tab returns as a standalone window on the current desktop.
- EC-4: Reopen history survives a client reload within the workspace → recent closes remain reopenable; exact retention depth is bounded and documented.

### US-009: Pin a long-lived tab

**As an** Operator, **I want** to pin tabs I keep for hours, **so that** they stay put and cannot be closed by reflex.

Acceptance criteria:

- AC-1: Given a tab, when I choose "Pin" from its context menu, then it moves to the left end of the deck, shrinks toward glyph-only, and shows no close affordance.
- AC-2: Given a pinned tab, when I press ⌘W on it, then nothing closes; closing requires "Unpin" first (the context menu says so).
- AC-3: Given pinned tabs, when the workspace restarts, then pins persist — position, state, and pin flag.
- AC-4: Given a pinned tab, when I drag it, then it reorders only within the pinned region.

Edge cases:

- EC-1: Pinning every tab in a frame → allowed; ⌘W becomes inert for the frame's tabs; closing the window is still possible via window close, with pinned tabs restorable via ⌘⇧T.
- EC-2: Unpinning → the tab rejoins the unpinned region at its boundary, regains its close affordance, and restores its full label.
- EC-3: Dragging a pinned tab out to a new window → allowed (explicit gesture); the new standalone window keeps the pin flag for when it groups again.

### US-010: Bulk-close via the tab context menu

**As an** Operator, **I want** "close others" and "close to the right", **so that** deck cleanup is one action instead of many.

Acceptance criteria:

- AC-1: Given a tab's context menu, when I choose "Close other tabs", then every unpinned tab except this one closes; pinned tabs survive.
- AC-2: Given "Close tabs to the right", then unpinned tabs after this one close in order; all closed tabs enter the reopen history (⌘⇧T restores them newest-first).
- AC-3: Given ⌥-click on a tab's close button, then it acts as "close other tabs" (the shortcut market users know).

Edge cases:

- EC-1: "Close others" when every other tab is pinned → no-op with the menu items disabled to say so beforehand.
- EC-2: Bulk-close including tabs hosting running sessions → sessions keep running; the dock continues signaling any needs-input among them.

## Per-tab navigation

### US-011: Drill in without spawning windows or tabs

**As an** Operator, **I want** each tab to own its navigation stack, **so that** going deeper never scatters my layout.

Acceptance criteria:

- AC-1: Given a root list in a tab (e.g., Tasks), when I open an item, then the same tab navigates deeper — no new window, no new tab — and the head shows the breadcrumb contract (back chevron, clickable parents, leaf).
- AC-2: Given a drilled-in tab, when I press ⌘[ (or the back chevron), then only this tab's stack pops; sibling tabs don't move.
- AC-3: Given several tabs each at different depths, when I switch among them, then each preserves its own position and history independently.

Edge cases:

- EC-1: ⌘[ at a tab's root → quiet no-op (nothing to pop), never closes the tab.
- EC-2: A drilled-in entity is deleted elsewhere while its tab is inactive → activating the tab shows the truthful state: a clear gone-notice with a path back to the parent list.
- EC-3: Deep-linking (palette "Go to tab") into a drilled tab → arrives at the tab's current depth, not its root.

### US-012: Tab labels follow where each tab is

**As an** Operator, **I want** the label to reflect each tab's current leaf, **so that** the deck reads as a map of where everything is.

Acceptance criteria:

- AC-1: Given a tab at an app's root, when I read it, then it shows the app name (mild echo with the head is accepted at root).
- AC-2: Given the tab drills to "Tasks / Nightly triage / Run #418", then the label becomes the leaf ("Run #418") with the root glyph retained.
- AC-3: Given navigation back up, then the label follows in real time.

Edge cases:

- EC-1: Leaf names that collide across tabs ("Run #418" in two instances) → tooltips disambiguate with the full path; labels stay leaf-only.
- EC-2: There is no manual rename — labels are always derived; any rename affordance is absent by design (prevents label drift from the tab's truth).

## Drag & merge

### US-013: Drag a tab out into its own window

**As an** Operator, **I want** to tear a tab off the deck, **so that** I can promote a surface back to a full window at the pointer.

Acceptance criteria:

- AC-1: Given a multi-tab frame, when I drag a tab beyond the deck and release, then a standalone window appears at the pointer hosting that surface with its full navigation stack.
- AC-2: Given the origin frame, when the tab leaves, then the deck updates (or unmounts at one remaining tab per US-002) and the origin keeps its active-tab logic.
- AC-3: Given a drag in progress, when I press Escape (or drop back onto the origin deck), then the operation cancels and the deck returns to its prior order.

Edge cases:

- EC-1: Dragging out the active tab → origin activates its neighbor; the torn-off window is focused.
- EC-2: Dragging out of a tiled pane's deck → the new window floats at the pointer; the tiled group stays valid (no empty slot).
- EC-3: Drag interrupted by a client disconnect/reload → no half state: the tab is either still in the deck or a window, never both or neither.

### US-014: Drag a window onto a deck to merge — visibly and cancelably

**As an** Operator, **I want** unmistakable feedback when a drag would merge, **so that** accidental swallowing (the classic Mac complaint) cannot happen.

Acceptance criteria:

- AC-1: Given I drag a window over another window's deck, then a clear drop affordance appears (insertion point in the tab row) before anything commits; dropping elsewhere never merges.
- AC-2: Given I release on the affordance, then the dragged window's tab(s) join at the shown insertion point and become active; given I move away, the affordance disappears and no merge occurs.
- AC-3: Given a merge I regret, when I drag the tab back out (US-013) or press ⌘⇧T-equivalent undo path, then I recover the standalone window.

Edge cases:

- EC-1: Dropping on the deck of a window on another desktop (via overview surfaces, when exposed) → tab lands on that desktop; otherwise merging is desktop-local.
- EC-2: Drag-merge onto a frame whose deck is scrolled → the insertion point is visible at the drop position; the row auto-scrolls at the edges.
- EC-3: Merging a pinned standalone window (window that was a pinned tab before) → joins the pinned region per US-009.

### US-015: Merge and move without dragging

**As an** Operator, **I want** menu equivalents for merge and split, **so that** grouping never requires pointer precision.

Acceptance criteria:

- AC-1: Given the current desktop, when I invoke "Merge all windows" (tab/window context menu or palette), then that desktop's windows combine into one tabbed frame, order stable, focused window's tab active.
- AC-2: Given a tab, when I invoke "Move tab to new window", then it becomes a standalone window (menu twin of US-013).
- AC-3: Given both verbs, when I search the palette, then they appear with their current applicability (disabled states explained).

Edge cases:

- EC-1: "Merge all" with one window → no-op, communicated in the disabled menu item.
- EC-2: "Merge all" with minimized windows → minimized ones join the frame as tabs (and are no longer minimized) — stated behavior, no silent skips.
- EC-3: "Merge all" including pinned tabs from source frames → pins survive into the merged deck's pinned region.

## Multi-instance

### US-016: Open a second instance of an app on purpose

**As an** Operator, **I want** two independent views of the same app, **so that** I can, e.g., watch a running task in one tab while browsing the catalog in another.

Acceptance criteria:

- AC-1: Given Tasks is open, when I choose "Open new instance" (context menu per US-006), then a second Tasks surface opens (window or tab, per my choice) with its own navigation stack, filters, and view state.
- AC-2: Given both instances, when I change filters/views in one, then the other is untouched.
- AC-3: Given both instances in one deck, when I read the tabs, then each shows its own leaf (per US-012) rather than "Tasks" twice at depth.

Edge cases:

- EC-1: Instance count has no arbitrary product cap; at absurd volume (100× probe) the deck scrolls and palette search remains the escape hatch.
- EC-2: Closing one instance never affects the other's state.
- EC-3: A deep link/external open targeting the app → lands on the most recently used instance (never spawns a duplicate implicitly).

### US-017: Launch surfaces focus first, cycle on repeat

**As an** Operator, **I want** dock/palette activation to focus what exists, **so that** instances multiply only when I ask.

Acceptance criteria:

- AC-1: Given one instance open (as tab or window, any desktop), when I activate the app from the dock or palette, then that instance's window focuses, its tab activates, and the desktop switches if needed.
- AC-2: Given multiple instances, when I activate the app repeatedly, then activation cycles through instances in a stable order.
- AC-3: Given zero instances, when I activate the app, then a new window opens (default policy).

Edge cases:

- EC-1: The only instance is a tab in an unfocused group on another desktop → one activation gets me there (desktop switch + focus + tab activation), no intermediate stops.
- EC-2: Cycling while an instance is minimized → the minimized one un-minimizes when its turn arrives.

## Attention & status

### US-018: Read every agent's state from the deck without switching

**As a** Supervisor, **I want** live state on each tab, **so that** the deck works as my attention queue.

Acceptance criteria:

- AC-1: Given tabs hosting sessions, when a session is running, then its tab shows the running state (pulsing dot); when it needs me, an accent attention badge (count) appears; idle/done states show their quiet forms.
- AC-2: Given a state change on an inactive tab, when it happens, then the tab updates live without my interaction.
- AC-3: Given the signal palette rule, when I scan the deck, then accent color appears only for genuine state/attention — selection stays neutral (no decorative color).

Edge cases:

- EC-1: A tab compressed to minimum width or scrolled out of view → its attention badge remains visible (badge survives compression; scrolled-out attention is still surfaced by the dock aggregate per US-019 and palette).
- EC-2: Many simultaneous needs-input states → each tab badges individually; no aggregate "everything is urgent" wash.
- EC-3: State flapping (running↔needs-input rapidly) → the tab shows the latest state without flicker storms.

### US-019: Never lose a needs-input signal by closing a tab

**As a** Supervisor, **I want** attention signals to outlive tabs, **so that** closing a view never silences an agent.

Acceptance criteria:

- AC-1: Given a needs-input session whose tab I close, when I look at the dock, then the sessions indicator still shows the pending-attention badge and the sessions list still surfaces it.
- AC-2: Given the dock app indicator, when any window/tab of an app exists, then the app's running indicator is lit; when the last one closes, it clears.
- AC-3: Given a dock activation on an app with a needs-input instance, when I click, then I land on the instance that needs me first.

Edge cases:

- EC-1: Needs-input arrives for a session with no tab anywhere → dock badge + sessions list carry the signal alone; opening from there works as any launch (US-006/US-017).
- EC-2: Multiple clients: a signal answered on one client → clears everywhere live.

## Keyboard & palette

### US-020: Navigate tabs entirely from the keyboard

**As an** Operator, **I want** the standard tab keyboard contract, **so that** my hands never leave the keys.

Acceptance criteria:

- AC-1: Given a focused multi-tab frame: ⌘T new tab (US-005) · ⌘W close (US-007) · ⌘⇧T reopen (US-008) · ⌃Tab / ⌃⇧Tab cycle MRU-adjacent · ⌘1–9 jump to tab by position (⌘9 = last tab) · ⌘[ pops the active tab's stack (US-011).
- AC-2: Given the shortcuts settings surface, when I view window-manager shortcuts, then the configurable tab actions appear with their defaults and are rebindable like existing actions, with conflicts flagged; the established ⌘[ navigation chord remains fixed.
- AC-3: Given a single-tab window, when I use tab shortcuts, then they degrade sanely: ⌘W closes the window, ⌃Tab/⌘1–9 are no-ops, ⌘T mounts the deck.

Edge cases:

- EC-1: Shortcut pressed while an input field is focused inside the content → editing keys are not stolen; the established editable-target guard applies.
- EC-2: ⌘1–9 with more than 9 tabs → 1–8 positional, 9 = last; tabs beyond reach remain accessible via cycling and palette.
- EC-3: User rebinds ⌃Tab away → the action remains available via its new chord and the palette; no hardcoded fallback fights the override.

### US-021: Jump to any tab anywhere from the palette

**As a** Supervisor, **I want** ⌘K to search open tabs across all windows and desktops, **so that** no tab is ever stranded by overflow.

Acceptance criteria:

- AC-1: Given the palette, when I type a tab's title/leaf/app/session name, then matching open tabs appear as a distinct result group ("Go to tab"), each showing its window/desktop context.
- AC-2: Given I select a result, then the right desktop activates, the window focuses, and the tab becomes active — one action total.
- AC-3: Given the palette's tab results, when a tab needs input, then its attention state is visible in the result row.

Edge cases:

- EC-1: Two tabs with identical labels → results disambiguate with full path/window context.
- EC-2: The target tab is in a minimized window → selection un-minimizes and lands correctly.
- EC-3: Zero open tabs matching → the group is absent; normal palette results (apps, sessions) still answer the query (no dead end).

## Persistence

### US-022: Everything is back after a restart

**As an** Operator, **I want** groups, order, pins, and per-tab positions to survive restarts, **so that** I can invest in arranging tabs.

Acceptance criteria:

- AC-1: Given tabbed frames across desktops, when the daemon and client restart, then every group returns: membership, tab order, pinned flags, each tab's surface and navigation stack, and window geometry.
- AC-2: Given each restored group, when it first renders, then its recorded last-active tab is active.
- AC-3: Given a fresh client joining an existing workspace, when it connects, then it sees the same groups as every other client (per-client active tab starts from the recorded last-active).

Edge cases:

- EC-1: Restore where a tab's surface no longer resolves (deleted session/entity) → the tab opens on the nearest truthful state with a clear notice (mirror of US-008.EC-2); the group itself always restores.
- EC-2: Persisted arrangement from an older shape (pre-tabs) → discarded once at load with a logged warning; the workspace reinitializes with a fresh layout (sessions, tasks, and runtime data untouched); no dual-format support lingers (greenfield hard-cut).
- EC-3: Restore at 100× volume (dozens of groups, hundreds of tabs) → the shell remains responsive; decks virtualize/scroll rather than degrade.

### US-023: Two screens, two active tabs, one shared truth

**As an** Operator, **I want** each client to keep its own active tab, **so that** my second monitor/browser doesn't fight my first.

Acceptance criteria:

- AC-1: Given the same workspace open in two clients, when I switch tabs in client A, then client B's active tab is unchanged.
- AC-2: Given client A adds/closes/reorders/pins a tab, then client B reflects the membership change live (shared durable truth).
- AC-3: Given both clients, when each switches tabs, then the group's recorded last-active updates (most recent wins) — affecting only future restores, never the other live client.

Edge cases:

- EC-1: Client B has active a tab that client A closes → client B's frame activates the close-policy neighbor live.
- EC-2: A client disconnects and returns → its own active-tab choices are re-derived (last-active fallback); membership is exactly the shared state.

## Tiled integration

### US-024: Tabbed panes inside tiled layouts

**As an** Operator, **I want** a tiled pane to hold a tabbed group with its own deck, **so that** tabs and splits compose — and stacked windows finally have a switcher.

Acceptance criteria:

- AC-1: Given a tiled layout, when a pane holds ≥2 windows (via the existing stack arrangement or by merging), then that pane renders its own deck and behaves per every deck rule (switch, close, drag out, pin).
- AC-2: Given the previously existing stacked arrangement, when this ships, then every stacked pane is reachable by pointer via its deck — the unreachable-member defect is gone.
- AC-3: Given multiple tabbed panes side by side, when I use each deck, then they operate independently (per-pane decks, per reference rule D6).

Edge cases:

- EC-1: Dragging the last non-active tab out of a tiled pane's deck → pane becomes a normal single-window tile (deck unmounts).
- EC-2: Resizing a narrow tiled pane with many tabs → the deck compresses/scrolls within the pane's width; layout handles stay usable.
- EC-3: Layout undo/redo across a grouping change → topology history restores group membership consistently with the rest of the layout step.

## Agent management & extensibility

### US-025: Inspect groups and tabs with structured output

**As an** Agent, **I want** to read the full tab topology, **so that** I can reason about the operator's screen before acting.

Acceptance criteria:

- AC-1: Given the window-listing command surfaces (CLI verb, HTTP/UDS routes, native tool), when I list windows, then group membership, tab order, active tab (last-active), pinned flags, and each tab's surface identity appear as structured fields.
- AC-2: Given structured output modes, when I request them, then tab data round-trips losslessly (machine-parseable, documented shape).
- AC-3: Given the live event/stream surface, when tabs change (open/close/move/activate/pin), then subscribed clients receive the change without polling.

Edge cases:

- EC-1: Listing a workspace with zero groups → fields present with empty values, not absent/null surprises.
- EC-2: A consumer on the old shape after the change ships → the shape is the new one only; the change is versioned/announced (greenfield hard cut, no dual output).

### US-026: Manage tabs with the same power a human has

**As an** Agent, **I want** commands to open, group, activate, move, pin, and close tabs, **so that** I can arrange the operator's workspace on request.

Acceptance criteria:

- AC-1: Given the semantic command surface, when I issue tab operations (open-as-tab, group windows, ungroup, activate, reorder, pin/unpin, close), then each succeeds atomically against the current revision and is visible to all clients live.
- AC-2: Given an invalid operation (unknown window, tab not in group, activating a non-member, stale revision), then I get the documented deterministic error code for that case — never a silent no-op or generic failure.
- AC-3: Given any tab operation, when I perform it via CLI, HTTP, or UDS, then behavior and payloads are identical across the three (parity), and the equivalent native tools exist for in-session agents.
- AC-4: Given the official Compozy skill's window-management reference, when tabs ship, then agents can discover these operations from it.

Edge cases:

- EC-1: Concurrent agent + human mutations on one group → revision checks serialize; the loser retries transparently or receives the stale-revision error.
- EC-2: Agent closes/moves the operator's active tab → operator's client follows the close policy live (no zombie frame).
- EC-3: Command referencing a tab that was just closed → deterministic not-found error (state transitions probe).
- EC-4: Malformed group specs (duplicate members, self-grouping, empty member list) → rejected with specific validation errors (invalid-input probe).

### US-027: React to tab lifecycle events

**As an** Automation author, **I want** tab changes on the existing hook/event surfaces, **so that** automations can respond to grouping, activation, and close.

Acceptance criteria:

- AC-1: Given the window-manager event family, when tabs are opened/closed/grouped/ungrouped/activated (durably), then corresponding typed events fire with workspace, window, and group identity in the payload.
- AC-2: Given hook introspection surfaces, when I list available events, then the tab events are discoverable with documented payloads.

Edge cases:

- EC-1: Per-client active-tab switches (presentation-only) → explicitly not a durable event (documented), so automations aren't spammed by every glance; the coalesced durable last-active commit is the observable record and fires the activation event.
- EC-2: Event storm from bulk operations (merge-all of 10 windows) → events arrive as a coherent batch tied to one revision change, not 10 interleaved partials.

### US-028: Saved layouts round-trip tab groups

**As an** Agent (or Operator), **I want** layout export/apply and layout profiles to carry groups, **so that** a saved arrangement restores tabs faithfully.

Acceptance criteria:

- AC-1: Given a workspace with tabbed frames, when I export the layout, then group membership, order, pins, and active-tab records appear in the exported document.
- AC-2: Given applying that document (same or another workspace), when it applies, then groups reconstruct exactly; validation rejects malformed group specs with specific errors.
- AC-3: Given existing layout profiles, when tabs ship, then profiles save/restore groups with no separate opt-in.

Edge cases:

- EC-1: Applying a pre-tabs layout document → rejected with the deterministic version error naming the unsupported version; re-exporting produces a current document (no converter, no legacy emulation shims).
- EC-2: Applying a document referencing surfaces that don't resolve in the target workspace → the standing layout-apply resolution rules cover tabs the same as windows (no special-case gap).

## Route views amendment

### US-029: Route views live in the strip everywhere

**As an** Operator, **I want** peer views (List | Kanban) in the toolbar strip on every route, **so that** the head always reads identity + status and never crowds.

Acceptance criteria:

- AC-1: Given any route with peer views, when I view it — single-tab window included — then the views selector renders as the strip's leading group (order: views · filters · spacer · display-mode) and is absent from the head.
- AC-2: Given a drilled-in tab, when the breadcrumb needs width, then it gets the head's full width (views no longer compete).
- AC-3: Given the canonical design-system documents, when this ships, then the amended rules (chrome rows, route-views placement) match the product.

Edge cases:

- EC-1: A route with views but no other toolbar content → the strip renders with views alone (no empty-strip special case).
- EC-2: A route with no peer views → no views group; strip renders only if other tools exist (unchanged).

## Edge-Case Sweep Notes

Every story above was probed against all ten classes (invalid input, empty/missing, limits, permissions, concurrency, interruption, repetition, ordering, state transitions, scale). Classes without a recorded EC for a story were probed and found not applicable at the product level, most commonly:

- **Permissions**: tabs are presentation topology inside one workspace; they carry no data of their own, so cross-workspace access is governed entirely by the existing session/workspace access rules of the surfaces the tabs host. Agent tab commands run under the same workspace scoping as today's window commands.
- **Repetition**: all tab commands are revisioned; identical retries after success are no-ops or deterministic errors (covered in US-026).
