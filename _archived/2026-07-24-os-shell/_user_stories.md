# User Stories: AGH OS Shell (`os-shell`)

Canonical behavior catalog for the AGH OS shell. Companion to `_prd.md`; consumed by
`_techspec.md` (component mapping) and `_tests.md` (coverage matrix).

## Personas

- **Operator** — a developer running real agent work on their machine: several concurrent agent sessions, tasks, and automations in one or more workspaces. Needs to start, supervise, resume, inspect, and repair work, and to trust that what the screen shows is what the runtime is doing.
- **Runtime developer** — an engineer extending AGH against daemon contracts. Needs every desktop capability to exist as a documented, structured surface — never a UI-only path — with stable identifiers and deterministic errors.
- **Agent** — a first-class non-human user. Operates AGH through structured surfaces (CLI with structured output, API) and must be able to inspect and manage the same state the desktop renders, including the desktop arrangement itself.

## Story Index

| ID | Feature Area | Persona | Story |
|---|---|---|---|
| US-001 | Desktop & windows | Operator | Open product surfaces as windows from the dock |
| US-002 | Desktop & windows | Operator | Arrange windows freely within the desktop |
| US-003 | Desktop & windows | Operator | Window lifecycle: zoom, minimize, restore, close |
| US-004 | Multi-session | Operator | Observe several live sessions side by side |
| US-005 | Desktop & windows | Operator | Navigate inside a window with a deepening breadcrumb |
| US-006 | Attention | Operator | See which apps need me from the dock |
| US-007 | Sessions rail | Operator | Scan and open sessions from the rail |
| US-008 | Attention | Operator | Resolve approvals from the menubar bell |
| US-009 | Palette & keyboard | Operator | Open anything with ⌘K |
| US-010 | Spaces | Operator | Each workspace is its own space |
| US-011 | Continuity | Operator | Everything is exactly where I left it |
| US-012 | Continuity | Operator | The same desktop follows me across tabs and devices |
| US-013 | Continuity | Operator | The shell keeps working when sync is unavailable |
| US-014 | Compact mode | Operator | Use the shell on a small viewport |
| US-015 | Appearance | Operator | Wallpaper and motion respect my preferences |
| US-016 | Deep links & URL | Operator | Every surface stays addressable and shareable |
| US-017 | Agent-managed desktop | Agent | Inspect and arrange the operator's desktop through structured surfaces |
| US-018 | Structured surfaces | Runtime developer | Desktop state is a documented, deterministic surface |
| US-019 | Menubar | Operator | Reach system actions from the menubar menus |
| US-020 | Desktop & windows | Operator | Dialogs belong to their window |
| US-021 | Desktop & windows | Operator | Snap windows to halves and quarters in one gesture |

## Desktop & windows

### US-001: Open product surfaces as windows from the dock

**As an** operator, **I want** every product surface (dashboard, tasks, agents, marketplace, …) to open as a window from the dock, **so that** my tools behave like applications on a desktop instead of one page at a time.

Acceptance criteria:

- AC-1: Given an empty desktop, when I click a dock item, then its window opens at a sensible default placement and receives focus.
- AC-2: Given an app window is already open, when I click its dock item again or select it in the palette, then the existing window is focused — no duplicate opens.
- AC-3: Given a window is open, when I look at the dock, then its item shows an open indicator.
- AC-4: Given an empty desktop, then a hint teaches me the two openers (palette and dock).

Edge cases:

- EC-1: Empty/missing — first run with no saved desktop → empty desktop with the hint; no window opens uninvited.
- EC-2: Invalid input — saved desktop references a surface that no longer exists → that entry is discarded silently; the rest of the desktop restores.
- EC-3: Limits — opening beyond the soft window guidance (12 open windows) → windows still open, and the shell surfaces a gentle notice suggesting minimize/spaces.
- EC-4: State transitions — clicking the dock item of a minimized window → the window restores and focuses (not a second instance).

### US-002: Arrange windows freely within the desktop

**As an** operator, **I want** to drag and resize windows within the desktop, **so that** I control my own working layout.

Acceptance criteria:

- AC-1: Given a floating window, when I drag it by its head, then it follows the pointer and stays fully reachable inside the desktop (menubar and dock never get covered or lost behind edges).
- AC-2: Given a floating window, when I resize from its edges or corners, then it resizes within its minimum size and the desktop bounds.
- AC-3: Given windows overlap, when I press one, then it comes to the front and shows focused styling; the others dim.

Edge cases:

- EC-1: Limits — dragging a window mostly off-screen → it clamps back to a reachable position.
- EC-2: Interruption — shrinking the browser window → open windows re-fit so none is stranded outside the visible desktop.
- EC-3: Concurrency — starting a drag while another window is animating → both complete independently; focus follows my pointer press.
- EC-4: Repetition — a fast sequence of moves → only the final resting position persists; no jitter on restore.

### US-003: Window lifecycle: zoom, minimize, restore, close

**As an** operator, **I want** macOS-style window controls (close / minimize / zoom) plus keyboard equivalents, **so that** managing windows costs no thought.

Acceptance criteria:

- AC-1: Given a floating window, when I press zoom, then it fills the desktop; pressing zoom again returns it to its exact previous size and position.
- AC-2: Given a window, when I minimize it, then it animates into its dock item and the dock shows a minimized indicator.
- AC-3: Given a minimized window, when I restore it, then its content is immediately present (no blank reload feel) and it regains focus.
- AC-4: Given a focused window, keyboard close and minimize work; every window action is also reachable from the palette.

Edge cases:

- EC-1: Interruption — minimizing a session window while its agent streams → nothing is lost; on restore the transcript shows everything that happened meanwhile.
- EC-2: State transitions — closing the focused window → the next-front window gains focus; closing the last window → empty desktop with hint.
- EC-3: Invalid input — the browser reserves a shortcut (e.g. close-tab) → the shell does not fight it; the palette lists the same action as the reliable path, and this limitation is documented.
- EC-4: Repetition — double-pressing minimize rapidly → one minimize; no stuck intermediate state.

### US-005: Navigate inside a window with a deepening breadcrumb

**As an** operator, **I want** lists and details to open inside the same app window with a breadcrumb that deepens, **so that** drilling into a task or agent never disturbs the rest of my desktop.

Acceptance criteria:

- AC-1: Given the tasks window shows the list, when I open a task, then the detail renders inside the same window and the breadcrumb shows the deeper trail.
- AC-2: Given a window shows a detail, when I use the breadcrumb, then I return to the shallower level inside that window.
- AC-3: Given I navigate inside the focused window, then the browser URL follows, and browser back returns me exactly one step.

Edge cases:

- EC-1: Invalid input — a detail link for an entity that no longer exists → the window shows a truthful error state with a path back to the list; the window itself stays healthy.
- EC-2: Ordering — browser back past the window's first location → focus returns to the previously focused window (focus history), never a blank page.
- EC-3: State transitions — the entity shown is deleted elsewhere while open → the window reflects the deletion truthfully instead of freezing stale content.

## Multi-session

### US-004: Observe several live sessions side by side

**As an** operator, **I want** each session to open in its own window, **so that** I can watch several agents work simultaneously — the thing a single-page app can never give me.

Acceptance criteria:

- AC-1: Given two running sessions, when I open both, then two independent session windows stream live transcripts at the same time.
- AC-2: Given multiple session windows, when I reply in one, then the others keep their scroll position and state untouched.
- AC-3: Given a session already has a window, when I open that session again from anywhere (rail, palette, link), then its existing window is focused.

Edge cases:

- EC-1: Scale — many session windows open → the shell stays responsive; minimized session windows cost nothing visible; the soft guidance (US-001.EC-3) applies.
- EC-2: State transitions — a session reaches a terminal state while its window is open → the window shows the truthful terminal state and stops implying liveness.
- EC-3: Interruption — the daemon connection drops mid-stream → the window surfaces the disconnection honestly and recovers when the daemon returns.
- EC-4: Permissions — a session belonging to another workspace → opening it follows the workspace-switch rule (US-016.EC-2), never a cross-workspace leak in place.

## Attention

### US-006: See which apps need me from the dock

**As an** operator, **I want** dock badges and indicators that reflect real runtime state, **so that** a glance tells me where attention is needed.

Acceptance criteria:

- AC-1: Given sessions are waiting on me, then the sessions dock item shows the waiting-session count (sourced from the live session catalog); given tasks await approval, then the tasks dock item shows the awaiting-approval count (sourced from the tasks dashboard) — each badge names a real runtime projection, never a derived guess.
- AC-2: Given nothing needs attention, then no badge renders (zero is invisible, never "0").
- AC-3: Given state changes (a session starts waiting, a task lands for approval), then badges update live without a reload.

Edge cases:

- EC-1: Scale — a count beyond badge width → the badge caps its display (e.g. "9+") without lying about zero vs non-zero.
- EC-2: Empty/missing — a badge's source is briefly unavailable or stale → that badge hides rather than showing a stale or invented number.

### US-008: Find everything waiting on me under the menubar bell

**As an** operator, **I want** everything waiting on me — sessions waiting for input or approval, tasks awaiting approval — aggregated under a menubar bell, **so that** finding the next thing that needs me never requires hunting through windows.

Acceptance criteria:

- AC-1: Given waiting items exist, then the bell shows their count; opening it lists each item with its context (what is waiting, where, since when) — every item backed by a live runtime projection.
- AC-2: Given a listed item, when I select it, then the owning window opens/focuses exactly where the runtime-backed approve/deny (or reply) surface lives; resolving it there removes the item from the bell live.
- AC-3: Given nothing is waiting, then the bell popover states that nothing is waiting.

Edge cases:

- EC-1: Concurrency — an item is resolved elsewhere (another client, the CLI) → the list updates live; selecting a just-resolved item lands on the owning window showing its truthful current state, never a phantom pending item.
- EC-2: Interruption — the popover is open when the daemon connection drops → the popover reflects the disconnect instead of listing items it cannot vouch for.

## Sessions rail

### US-007: Scan and open sessions from the rail

**As an** operator, **I want** a sessions rail with recent and grouped views, live status, and filtering, **so that** sessions — the heart of AGH — stay one glance away.

Acceptance criteria:

- AC-1: Given the rail is open, then recent sessions show title, agent, status, and age; a distinct status vocabulary separates running / waiting / failed / done / idle.
- AC-2: Given many sessions, when I switch to the full view, then sessions group by agent with collapsible groups that remember their state.
- AC-3: Given the filter, when I type, then the list narrows by title or agent live.
- AC-4: Given a rail row, when I click it, then that session's window opens or focuses.
- AC-5: Given the dock's sessions item or the palette's rail action, when I trigger it, then the rail toggles open/closed; its state persists with my desktop.

Edge cases:

- EC-1: Empty/missing — no sessions yet → an empty state explains what creates sessions and offers starting one.
- EC-2: Scale — hundreds of sessions → the rail stays scrollable and filterable; grouping keeps it navigable.
- EC-3: State transitions — a listed session ends → its row updates status live; opening it shows the terminal session truthfully.
- EC-4: Limits — in compact mode the rail presents as a full-height overlay sheet; opening it never hides the focused window's state permanently (dismiss returns exactly where I was).

## Palette & keyboard

### US-009: Open anything with ⌘K

**As an** operator, **I want** a global command palette on the canonical shortcut, **so that** navigation costs zero screen space and zero pointer travel.

Acceptance criteria:

- AC-1: Given any focus context, when I press ⌘K, then the palette opens listing apps, sessions, and shell actions, filtered as I type.
- AC-2: Given a result, when I confirm it, then the corresponding window opens or focuses and the palette closes.
- AC-3: Given a form or dialog that contains a visible, enabled RuntimeSelector, when focus is inside that host and I press ⌘J, then the runtime picker opens (its previous shortcut having been ceded to the palette), with its visible hint updated. An active session composer does not expose the picker because the runtime cannot change an established session's runtime per prompt.

Edge cases:

- EC-1: Empty/missing — a query with no matches → an explicit "no matches" state; Escape closes.
- EC-2: Ordering — palette opened while an overlay is already up → the palette wins cleanly; Escape unwinds one layer at a time.
- EC-3: Interruption — palette works in degraded mode (US-013) — navigation never depends on sync availability.

## Spaces

### US-010: Each workspace is its own space

**As an** operator, **I want** each workspace to own an independent desktop arrangement — its space — **so that** switching projects swaps my whole bench, not just a filter.

Acceptance criteria:

- AC-1: Given windows arranged in workspace A, when I switch to workspace B, then B's space appears (its own windows and wallpaper) and A's arrangement is preserved untouched.
- AC-2: Given the Spaces overview, then each workspace shows a thumbnail of its arrangement; choosing one switches to it.
- AC-3: Given a workspace never used before, then its space starts empty with the desktop hint.

Edge cases:

- EC-1: State transitions — a workspace is deleted → its space and saved arrangement disappear with it; no orphan state resurfaces later.
- EC-2: Interruption — switching spaces while sessions stream in the old space → the switch is clean; returning shows those windows live again with nothing lost.
- EC-3: Scale — many workspaces → the overview stays scannable (thumbnails + names), and workspace switching remains available from the menubar and palette.

## Continuity

### US-011: Everything is exactly where I left it

**As an** operator, **I want** the desktop to restore exactly — windows, positions, sizes, minimized state, focus, and each window's inner location — **so that** reopening AGH is resumption, not reconstruction.

Acceptance criteria:

- AC-1: Given an arranged desktop, when I reload the page, then every window returns to its exact place and state, and the focused window is focused again.
- AC-2: Given the daemon restarts, when the shell reconnects, then the desktop state survives (it belongs to the daemon, not the page).
- AC-3: Given a window was showing a deep location (a specific task's detail), then restoration returns it to that exact location.

Edge cases:

- EC-1: Invalid input — saved state that is unreadable or from an unknown future version → that entry is dropped gracefully; the desktop still boots with everything salvageable.
- EC-2: Empty/missing — partial saved state (some windows valid, one not) → best-effort restore of the valid ones.
- EC-3: Repetition — restoring twice in a row (double reload) → identical result; restoration is idempotent.

### US-012: The same desktop follows me across every client of my daemon

**As an** operator, **I want** desktop changes to converge live across every client connected to my daemon (tabs, browsers, machines pointing at it) viewing the same workspace, **so that** the desktop feels like one place, not per-tab copies.

Acceptance criteria:

- AC-1: Given the same workspace open in two tabs, when I move a window in one, then the other reflects the move live.
- AC-2: Given two clients disagree (both moved the same window), then both settle on one truthful final state — no flicker loop, no split-brain.
- AC-3: Given a client was offline, when it reconnects, then it adopts the current desktop truth.

Edge cases:

- EC-1: Concurrency — simultaneous edits to different windows → both changes survive (per-window independence).
- EC-2: Interruption — a tab closes mid-change → the last completed change wins; no corruption.
- EC-3: Scale — several clients on one workspace → updates stay smooth; the desktop never thrashes.

### US-013: The shell keeps working when sync is unavailable

**As an** operator, **I want** full desktop functionality even when persistence/sync is down, **so that** a degraded backend never blocks my work.

Acceptance criteria:

- AC-1: Given sync is unavailable at load, then the shell boots with a working in-memory desktop and a non-blocking indicator states the degraded condition truthfully.
- AC-2: Given degraded mode, when sync recovers, then the indicator clears and persistence resumes without a reload — and the arrangement I built while degraded is pushed and wins for the windows I touched ("what I see is what persists"); everything I didn't touch adopts the daemon's truth.
- AC-3: Given degraded mode, all window operations, navigation, palette, and live session content keep working (only layout persistence is affected).

Edge cases:

- EC-1: Interruption — sync drops mid-session → the indicator appears; edits continue locally; on recovery they persist per AC-2.
- EC-2: Repetition — repeated flaps (down/up/down) → the indicator reflects current truth without spamming notifications, and each recovery applies AC-2 consistently.

## Compact mode

### US-014: Use the shell on a small viewport

**As an** operator, **I want** the shell to adapt below the desktop breakpoint instead of blocking me, **so that** a narrow browser window or small screen still gives me AGH.

Acceptance criteria:

- AC-1: Given a viewport below the desktop breakpoint (960px), then open windows present as a full-viewport stack and the dock becomes a bottom tab bar (scrollable, no magnification, safe-area aware) — same windows, same state, no floating geometry; the zoom control disappears (meaningless in a stack) and window controls grow comfortable touch targets.
- AC-2: Given compact mode, when the viewport grows past the breakpoint, then floating windows return with their exact prior geometry.
- AC-3: Given compact mode, deep links, palette, sessions, and attention surfaces all keep working.

Edge cases:

- EC-1: Interruption — crossing the breakpoint mid-drag → the gesture ends safely; no half-applied geometry.
- EC-2: Ordering — arriving via deep link while compact → the target window opens focused in the stack; going wide later floats it at its saved or default placement.

## Appearance

### US-015: Wallpaper and motion respect my preferences

**As an** operator, **I want** per-space wallpapers and motion that honors reduced-motion preferences, **so that** the desktop is mine and never hostile.

Acceptance criteria:

- AC-1: Given the appearance settings, when I pick a wallpaper, then the current space changes and the choice persists with that space.
- AC-2: Given my system prefers reduced motion (or I enable the in-product toggle), then window, dock, and minimize animations reduce to instant or crossfade equivalents.
- AC-3: Given dock magnification is off, then dock icons stay static.

Edge cases:

- EC-1: Permissions/system — the system-level reduced-motion preference always wins over the in-product toggle.
- EC-2: Empty/missing — no saved appearance → the default wallpaper and full motion apply.

## Deep links & URL

### US-016: Every surface stays addressable and shareable

**As an** operator, **I want** the browser URL to always identify what I'm focused on, **so that** links, bookmarks, and back/forward keep their meaning on a desktop.

Acceptance criteria:

- AC-1: Given any focused window, then the URL reflects its current location; sharing that URL opens the same surface focused.
- AC-2: Given a deep link into a detail (e.g. a specific task), then the owning app's window opens focused at that exact location.
- AC-3: Given focus moves between windows, then browser back re-focuses the previous window; forward re-applies.

Edge cases:

- EC-1: Invalid input — a link to an entity that no longer exists → the owning window opens with a truthful error state (US-005.EC-1).
- EC-2: Ordering — a link into another workspace's content → the shell switches to that workspace's space and opens the target there; nothing renders cross-workspace in place.
- EC-3: Repetition — opening the same deep link twice → one window, focused twice (idempotent).

## Agent-managed desktop

### US-017: Inspect and arrange the operator's desktop through structured surfaces

**As an** agent, **I want** to read and write the desktop arrangement through the same structured surfaces I use for everything else, **so that** the desktop is manageable state, not a human-only UI.

Acceptance criteria:

- AC-1: Given a workspace, when I list its desktop state through the CLI with structured output, then I receive every entry with stable identifiers and versions.
- AC-2: Given a valid write (e.g. repositioning a window), then any open desktop for that workspace reflects the change live.
- AC-3: Given an invalid write (oversized payload, unknown workspace, stale version guard), then I receive a deterministic, documented error — and the operator's desktop is unaffected.

Edge cases:

- EC-1: Invalid input — a structurally broken payload written to a desktop key → the shell ignores what it cannot read and keeps functioning; the daemon never validates presentation semantics on the agent's behalf.
- EC-2: Limits — writes beyond the per-entry size or per-workspace quota → rejected with the documented error identifiers.
- EC-3: Concurrency — an agent and the operator move the same window near-simultaneously → last write wins cleanly; the version counter makes the ordering observable.
- EC-4: Permissions — a write against a nonexistent or deleted workspace → deterministic not-found error; nothing is created implicitly.

## Structured surfaces

### US-018: Desktop state is a documented, deterministic surface

**As a** runtime developer, **I want** the desktop-state surface documented with stable routes, verbs, error identifiers, and scope rules, **so that** I can build on it without reading the web app's source.

Acceptance criteria:

- AC-1: Given the generated references, then the desktop-state verbs and routes appear with their request/response shapes and error identifiers.
- AC-2: Given the docs, then the state's scoping rule (workspace-scoped, presentation-only, distinct from agent memory) and its intentionally desktop-only v1 surface are stated explicitly.
- AC-3: Given the CLI and the API, then both expose the same state with identical semantics (parity is observable).

Edge cases:

- EC-1: Empty/missing — desktop state with no entries → an empty list result, not an error.
- EC-2: State transitions — behavior on workspace deletion (state purged) is documented and observable.

## Menubar

### US-019: Reach system actions from the menubar menus

**As an** operator, **I want** the menubar's identity and menus (workspace trigger, Session, View, Help, settings, logo), **so that** system-level actions have one predictable home.

Acceptance criteria:

- AC-1: Given the menubar, then the workspace trigger shows the active workspace and opens the switcher; the logo focuses/opens the dashboard window; the cog opens the settings window.
- AC-2: Given the Session menu, then "New session" (⌘N) and "New task" work; given the View menu, then "Spaces overview" (⇧⌘S) and appearance shortcuts work; given Help, then the shortcut reference is reachable.
- AC-3: Given any open menu or popover, then Escape closes one layer at a time and focus returns to the trigger.

Edge cases:

- EC-1: Empty/missing — a menu action whose target is unavailable (e.g. new session with no agents configured) → the action leads to the truthful empty/onboarding state, never a dead click.
- EC-2: Concurrency — two popovers (menu + bell) → opening one closes the other; no stacked orphan popovers.
- EC-3: Compact — menubar menus collapse below the breakpoint; every menu action remains reachable via the palette.

### US-020: Dialogs belong to their window

**As an** operator, **I want** dialogs and sheets to stay inside the window that opened them, **so that** a confirmation in one app never freezes the agents I'm watching in another.

Acceptance criteria:

- AC-1: Given a dialog opened from window content (confirm, form, sheet), then it renders centered inside that window, dims only that window, and moves with it when dragged.
- AC-2: Given a dialog open in window A, when I click or type in window B, then B responds normally; returning to A shows the dialog exactly as left.
- AC-3: Given a flow that is a route (e.g. creating a task), then it opens as a deeper location inside its app's window with the breadcrumb reflecting it — shareable like any other location.
- AC-4: Given desktop-level surfaces (palette, Spaces overview, toasts, onboarding, workspace-destroying confirms), then only those may cover the desktop; nothing else does.

Edge cases:

- EC-1: Interruption — minimizing a window with an open dialog → the window minimizes but the dialog's unsaved state survives; restoring shows it untouched.
- EC-2: Limits — a dialog larger than its window → it clamps to the window bounds with internal scrolling, never overflowing onto the desktop.
- EC-3: Ordering — window A's dialog never paints above focused window B; the dialog obeys its window's stacking position.
- EC-4: Scale — compact mode → the window fills the viewport, so its dialogs naturally present full-viewport; dismissal returns to the same window state.

## Window snap (ADR-009 amendment)

### US-021: Snap windows to halves and quarters in one gesture

**As an** operator, **I want** to throw a window against a desktop edge or corner — or use the palette/keyboard — and have it fill that half or quarter, **so that** building a side-by-side supervision layout takes one gesture instead of manual drag-and-resize.

Acceptance criteria:

- AC-1: Given a drag toward the left/right desktop edge, then a zone overlay previews the half during the drag and release snaps the window to it; corners preview and snap quarters; releasing outside every zone changes nothing.
- AC-2: Given a snapped window, then its geometry is proportional — resizing the browser viewport keeps it filling the same fraction of the desktop, without writing new state.
- AC-3: Given a snapped window, when I drag it away from its zone, then it detaches and restores its pre-snap size; zooming a snapped window unsnaps it (one derived geometry at a time).
- AC-4: Given the palette (and, where the browser allows, the shortcuts), then every snap action and restore is reachable without a pointer.
- AC-5: Given a snap in one client, then every client of the same daemon shows the window snapped to the same fraction of its own viewport; an agent writing snap fractions arranges the desktop live.

Edge cases:

- EC-1: Interruption — Escape during a snap drag cancels the preview and leaves geometry unchanged.
- EC-2: Limits — snapping is unavailable in compact presentation (actions no-op, no affordance renders); a zone that would derive below the window minimum clamps to the minimum.
- EC-3: Concurrency — an operator drag and an agent snap write racing on the same window settle last-writer-wins by version order with no oscillation.
- EC-4: Reduced motion — the zone overlay appears and disappears without fade/morph animation; the affordance itself never disappears.
