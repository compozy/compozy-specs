# TechSpec: Window Tabs (the Deck)

Implements `.compozy/tasks/window-tabs/_prd.md`. Product decisions live in ADR-001..007; technical decisions in ADR-008..013.

**MVP Boundary:** the MVP implements the full PRD v1 scope — deck UI over tiled and floating tab groups, multi-instance apps, per-tab navigation stacks, frictionless close + multi-level reopen, pins, ⌘K tab search, menu merge/move, context-menu destinations, the D3 route-views-to-strip amendment, snapshot v3 with boot discard, and complete agent manageability (commands, CLI verbs, native tools, hooks, docs) — followed by the standard trailing QA pair. Out of scope (PRD Non-Goals): hover peek, live sub-labels, ephemeral preview tabs, a global tab-vs-window preference, manual renaming, cross-workspace tab sharing, small-screen deck behavior, and app-subtree virtualization (inactive tabs stay mounted).

## Executive Summary

Tabs are grouped windows (ADR-001): the tiled side reuses the existing `NodeKindStack` unchanged, and floating frames gain a sibling `FloatingStack` container on `Desktop` (ADR-008). New durable state is confined to the window-manager aggregate: `Window.NavStack` + `Window.Pinned`, `Desktop.FloatingStacks`, `Snapshot.ClosedEntries`, and a coalesced durable last-active per stack; per-client active tab rides the in-memory `ClientView` (ADR-009). Multi-instance apps come from deleting the web's deterministic `osWindowId` singleton scheme in favor of generated window IDs plus semantic focus-first lookup (ADR-010). The snapshot version bumps 2→3 as a hard cut: stale snapshots are discarded and reinitialized at load, and no converter ships for v2 layout profiles (ADR-012). **No fallback, no compat shim, no placeholder:** every delete target below is removed in the same change that replaces it; no dual decode paths, no legacy ID scheme, no version-2 acceptance anywhere.

The command surface grows by five semantic commands (`window.stack.group`, `window.stack.reorder`, `window.stack.set_active`, `window.pin`, `window.reopen`) and three payload extensions (`NavigateWindowCommand.Mode`, `CloseWindowCommand.Scope`, `OpenWindowCommand` stack target), each shipped end-to-end across HTTP/UDS/CLI/native tools/hooks/docs in one pass — the extended payloads included: `compozy window open --stack-target`, `compozy window navigate --mode`, `compozy window close --scope`, and the matching `compozy__window_{open,navigate,close}` native-tool schema updates co-ship with the new verbs (no HTTP/UDS-only capability). Zero new HTTP/UDS routes: everything flows through the existing preview/commands/stream/layout surface. The web refactors stack rendering from N overlapping hidden `<Rnd>` instances to one frame per group with a deck, rewrites window resolution to semantic lookup, and moves route views into the toolbar strip globally (D3/ADR-007).

## System Architecture

### Component Overview

| Component | Responsibility | Change |
| --- | --- | --- |
| `internal/windowmanager` (types, validate, normalize, reducers, history, clients) | Owns tab topology: stack nodes, floating stacks, nav stacks, pins, closed entries, per-client stack-active projection, coalesced last-active flush | Modified (no new package) |
| `internal/daemon` (repository, hooks wiring, native-tool bindings) | v3 discard-on-load, new hook event mapping, 5 new native tools | Modified |
| `internal/api/contract` + `internal/api/core` | Wire types for new fields/commands, History removed from wire snapshot, 2 new error codes | Modified |
| `internal/cli` | `compozy window group|activate|pin|unpin|reopen`, extended `window list` output, regenerated docs | Modified |
| `internal/config` | `[window_manager] nav_stack_limit`, `closed_entry_limit`; shortcuts allow-list gains tab actions | Modified |
| `internal/hooks` | 5 new window-manager events + introspection | Modified |
| `web/src/systems/os` | Deck UI, frame-based group rendering, semantic window lookup, tab shortcuts/palette/context menus, new-tab app, D3 strip migration | Modified |
| `packages/ui` | Finalize `context-menu` primitive (export + story + test); Topbar loses the `nav` slot, toolbar strip gains the leading views convention | Modified |

Data flow is unchanged: web/CLI/agents → contract command → `Manager.Execute` (CAS on revision) → reduce → normalize → validate → commit (clientstate CAS) → event frame broadcast → clients refetch snapshot (now History-free). Presentation flow: `window.focus` → `ClientView` (incl. new `StackActive`) → client frame to that client only.

### Architectural Boundaries

- **No new Go package.** All domain logic stays in `internal/windowmanager`; only `internal/daemon` wires (repository policy, hooks, native tools) per SD-008. `mage Boundaries` rules are untouched — no new `internal/api/*` subpackage.
- Import graph unchanged: `windowmanager` imports nothing new; `daemon` remains the sole multi-importer; `api/contract` stays free of `daemon`/`cli` imports; `cli` talks UDS only.
- Web: all changes inside `web/src/systems/os/` plus the two D3 call sites (`systems/os/apps/tasks`, `systems/marketplace`) and `packages/ui` (Topbar, context-menu). Cross-system imports stay barrel-only.
- The active-tab coalescer is Manager-internal, bound to the per-workspace lock (see Core Interfaces); no peer package may write `ActiveID` (authoritative-primitive exclusivity, L-005).
- **File-ownership split, decided before writing** (500-line cap; several targets are already near it):
  - `internal/windowmanager` new files: `floating_stack.go` (type + shared member helpers used by both container kinds), `closed_entries.go` (ClosedEntry push/consume/reservation set), `active_coalescer.go`, `reducer_stack_group.go`, `reducer_stack_reorder.go`, `reducer_stack_active.go`, `reducer_window_pin.go`, `reducer_window_reopen.go`; close scopes extend `reducer_window_close.go` (currently small).
  - `web/src/systems/os/runtime/`: new `window-manager-tab-commands.ts` owns tab command builders + dispatch (runtime files are at 477/441 lines and must not grow); new `lib/group-projection.ts` owns group/frame projection extracted from `layout-projection.ts`.
  - `packages/ui/src/index.ts` (495 lines): split into `index.ts` + `exports/<domain>.ts` files using named re-exports only (`export { A, B } from "./exports/menus"`), every file under the cap.

## Implementation Design

### Data Models

All state lives in the existing per-workspace snapshot blob (clientstate, domain `window_manager`, key `snapshot`) — **no SQL, no Goose/Atlas migration**. Side-table-vs-JSON decision: every new datum is a typed Go field on the domain structs serialized in the snapshot; JSON metadata bags are forbidden for all of it. The only raw-JSON leaf remains the pre-existing `RouteSearch map[string]json.RawMessage` (opaque router search params by nature). SQLite is not touched.

New/changed fields with rationale (marker 4):

- **SnapshotVersion** — `uint32 = 3`; hard cut, loader discards any other version (ADR-012).
- **Desktop.FloatingStacks** — `[]FloatingStack`; floating tab frames, sibling of `Groups`/`Floating` (ADR-008).
- **FloatingStack.ID** — `NodeID`; stable identity for deck, anchor, and hook references.
- **FloatingStack.WindowIDs** — `[]WindowID`; ordered members, pinned prefix first.
- **FloatingStack.ActiveID** — `*WindowID`; durable last-active, the restore seed (ADR-009).
- **FloatingStack.Rect** — `NormalizedRect`; frame rect — members' `FloatingRect` is ignored while grouped.
- **FloatingStack.Minimized** — `bool`; the frame minimizes as a unit.
- **Window.NavStack** — `[]RouteIntent`; navigation ancestors oldest-first, `Route` stays the current leaf (ADR-011). Length capped at write time by effective `nav_stack_limit`; absolute maximum 200 in validation.
- **Window.Pinned** — `bool`; pin survives regroup/ungroup/reopen (PRD US-009).
- **Snapshot.ClosedEntries** — `[]ClosedEntry`; bounded reopen history, newest last (ADR-013). Excluded from the history `State` (invariant 15); capped at write time by effective `closed_entry_limit`; absolute maximum 100 in validation.
- **ClosedEntry** — `{Windows []Window, ActiveID *WindowID, StackID *NodeID, DesktopID, Rect, ClosedAt}`; one window per tab close, N windows per frame close as a single entry.
- **ClientView.StackActive** — `map[NodeID]WindowID`; per-client active tab per stack, in-memory only (ADR-009).

Wire (contract) deltas: `WindowManagerWindow` gains `nav_stack`, `pinned`; `WindowManagerDesktop` gains `floating_stacks`; `WindowManagerSnapshot` **drops `history`** and gains `closed_entry_count int`; `WindowManagerClientView` gains `stack_active map[string]string`. `LayoutDocument` (v3) carries `FloatingStacks`, `NavStack`, `Pinned` (full round-trip, PRD BR-18) but never `ClosedEntries`.

### Core Interfaces

New domain types and command payloads (marker 3 — signatures final):

```go
// types.go
type FloatingStack struct {
    ID        NodeID         `json:"id"`
    WindowIDs []WindowID     `json:"window_ids"`
    ActiveID  *WindowID      `json:"active_id,omitempty"`
    Rect      NormalizedRect `json:"rect"`
    Minimized bool           `json:"minimized,omitempty"`
}

type ClosedEntry struct {
    Windows   []Window       `json:"windows"`
    ActiveID  *WindowID      `json:"active_id,omitempty"`
    StackID   *NodeID        `json:"stack_id,omitempty"`
    DesktopID DesktopID      `json:"desktop_id"`
    Rect      NormalizedRect `json:"rect"`
    ClosedAt  time.Time      `json:"closed_at"`
}

type NavigateMode string
const (
    NavigateReplace NavigateMode = ""        // default: today's overwrite semantics
    NavigatePush    NavigateMode = "push"
    NavigatePop     NavigateMode = "pop"
)
```

```go
// commands.go — new command IDs
const (
    CommandWindowStackGroup     CommandID = "window.stack.group"
    CommandWindowStackReorder   CommandID = "window.stack.reorder"
    CommandWindowStackSetActive CommandID = "window.stack.set_active"
    CommandWindowPin            CommandID = "window.pin"
    CommandWindowReopen         CommandID = "window.reopen"
)

type GroupWindowsCommand struct {          // bulk group: drag-merge, merge-all, agent setup (one revision, one hook)
    TargetWindowID WindowID   `json:"target_window_id"`         // defines the destination frame/pane
    WindowIDs      []WindowID `json:"window_ids"`                // ≥1, joins in payload order
    InsertIndex    *int       `json:"insert_index,omitempty"`    // insertion anchor in the target's member list; nil = append; clamped to the joiners' pin region
}
type ReorderStackCommand struct {
    WindowID WindowID `json:"window_id"`
    Index    int      `json:"index"`                             // clamped; pinned/unpinned regions enforced
}
type SetStackActiveCommand struct {        // durable last-active; presentation too when ClientID present
    WindowID WindowID `json:"window_id"`
}
type PinWindowCommand struct {
    WindowID WindowID `json:"window_id"`
    Pinned   bool     `json:"pinned"`
}
type ReopenCommand struct{}                // pops newest ClosedEntry; empty history = no-op success

// extended payloads
type NavigateWindowCommand struct {
    WindowID WindowID     `json:"window_id"`
    Route    RouteIntent  `json:"route,omitempty"`               // required for ""/push; MUST be absent for pop (strict decode rejects pop+route)
    Mode     NavigateMode `json:"mode,omitempty"`
}

type CloseScope string
const (
    CloseScopeTab    CloseScope = ""        // default: close this window/tab only
    CloseScopeGroup  CloseScope = "group"   // traffic-light frame close: every member, pinned included
    CloseScopeOthers CloseScope = "others"  // every unpinned member of the window's stack except this one
    CloseScopeRight  CloseScope = "right"   // unpinned members after this one in deck order
)
type CloseWindowCommand struct {
    WindowID WindowID   `json:"window_id"`
    Minimize bool       `json:"minimize,omitempty"`              // legal only with scope ""
    Scope    CloseScope `json:"scope,omitempty"`                 // one command = one revision = one ClosedEntry = one hook batch
}
type WindowSpec struct {
    ID           WindowID        `json:"id,omitempty"`           // now optional: daemon generates when empty
    App          string          `json:"app"`
    InstanceKey  *string         `json:"instance_key,omitempty"`
    Route        RouteIntent     `json:"route"`
    DesktopID    DesktopID       `json:"desktop_id"`
    FloatingRect NormalizedRect  `json:"floating_rect"`
    InsertTiled  bool            `json:"insert_tiled,omitempty"`
    StackTargetWindowID *WindowID `json:"stack_target_window_id,omitempty"` // open-as-tab: join target's frame
}
```

```go
// clients.go / manager internals
type ClientView struct {
    // ...existing fields...
    StackActive map[NodeID]WindowID `json:"stack_active,omitempty"`
}

// activeCoalescer buffers durable last-active updates per (workspace, stack) with an
// explicit lifecycle. The Manager today holds two lock layers (m.mu registry + per-workspace
// locks); the coalescer binds to the PER-WORKSPACE lock:
//   - noteStackActive: called under the workspace lock (focus/set_active paths); records the
//     pending value and (re)arms that workspace's timer (quiet period 2s).
//   - flushStackActiveLocked: caller ALREADY HOLDS the workspace lock — used synchronously
//     at the top of every durable Execute and inside UnregisterClient (which holds the lock).
//     A failed pre-durable flush FAILS the durable command (invariant 4); pending values are
//     retained for retry on the next trigger.
//   - timer callbacks run on the Manager's detached context (context.WithoutCancel of boot ctx),
//     acquire the workspace lock themselves, then call flushStackActiveLocked; timer-path
//     failures log a warning and retain pending values.
//   - Close(): stop accepting new notes (closed flag under m.mu), cancel all timers, acquire
//     each workspace lock, flush synchronously, and join the timer goroutines via the tracked
//     WaitGroup before returning. Lock order is always workspace lock → coalescer state; the
//     coalescer never takes m.mu while holding a workspace lock.
// Flush commits are history-exempt; they emit the window_manager.stack.activated hook (B-007)
// and a normal event frame. Manager-internal; no exported API.
type activeCoalescer struct {
    mu      sync.Mutex                                  // guards pending/timers/closed
    pending map[WorkspaceID]map[NodeID]WindowID
    timers  map[WorkspaceID]*time.Timer
    closed  bool
    wg      sync.WaitGroup
}
func (m *Manager) noteStackActive(ws WorkspaceID, stack NodeID, window WindowID)
func (m *Manager) flushStackActiveLocked(ctx context.Context, ws WorkspaceID) error
```

```go
// hooks (internal/hooks/events_window_manager.go — same family/payload shape)
const (
    HookWindowManagerWindowOpened   HookEvent = "window_manager.window.opened"
    HookWindowManagerWindowClosed   HookEvent = "window_manager.window.closed"
    HookWindowManagerStackGrouped   HookEvent = "window_manager.stack.grouped"
    HookWindowManagerStackUngrouped HookEvent = "window_manager.stack.ungrouped"
    HookWindowManagerStackActivated HookEvent = "window_manager.stack.activated" // durable activation: set_active + coalesced flush commits
)
// ChangeSet gains outcome fields the dispatcher maps to grouped/ungrouped:
type ChangeSet struct {
    // ...existing fields...
    StackGrouped   []NodeID `json:"stack_grouped,omitempty"`
    StackUngrouped []NodeID `json:"stack_ungrouped,omitempty"`
}
```

Reducer behavior contracts:

- `window.stack.group`: target's container decides shape — tiled leaf/stack → node stack (existing `DropCenter` machinery); floating window/stack → `FloatingStack`. Members are removed from their origins (`removeWindow`), placement set `stacked`, and spliced at `InsertIndex` (nil = append) in payload order; the index is clamped so pinned joiners land in the pinned prefix and unpinned joiners never precede it. Unknown target/member → `ErrWindowNotFound`; member == target or duplicates → `ErrInvalidCommand`. One command, one revision, one `ChangeSet`.
- `window.stack.reorder`: reorders within the member's stack; `Index` clamps to the member's region (pinned prefix vs rest); window not in a stack → `ErrNotStacked`.
- `window.stack.set_active`: member's stack `ActiveID` = window (both container kinds); not stacked → `ErrNotStacked`; with `ClientID` also sets that client's `StackActive`. History-exempt; emits `window_manager.stack.activated`.
- `window.pin`: sets `Pinned`, then re-collates the stack's pinned prefix (stable order). Pinning is legal on non-stacked windows (flag persists for future grouping).
- `window.close` scopes (all: non-minimize closes push exactly ONE `ClosedEntry`, cap `closed_entry_limit`, evict oldest; active-neighbor rule: when the durably active member closes, the next member to its right becomes active, else the nearest to its left):
  - scope `""`: pinned window → `ErrWindowPinned`; otherwise closes this window (1-window entry).
  - scope `"group"`: resolves the window's stack, closes ALL members (pinned included) as one entry preserving deck order + durable active; on a non-stacked window degrades to a plain close.
  - scope `"others"` / `"right"`: closes the unpinned members of the window's stack (all-but-this / strictly-after-this in deck order) as one multi-window entry; pinned members survive; on a non-stacked window or nothing to close → `Applied=false` success. `Minimize=true` with a non-empty scope → `ErrInvalidCommand`.
- `window.reopen`: pops the newest entry (consume-on-pop). For each entry window whose ID is still live, that window is skipped (stale after a layout undo); dead windows are recreated with original IDs, routes, nav stacks, pins. Rejoin `StackID` when it still exists (durable active restored); else frame entries rebuild a `FloatingStack` at `Rect` (single survivor → plain floating). Missing desktop → active desktop. Every-window-live or empty history → `Applied=false` success.
- `window.navigate` modes per ADR-011; `pop` with a non-empty `route` → `ErrInvalidCommand` (strict decode); pop on empty stack → applied=false success.
- `window.move` `DropCenter` extends to floating targets (forms/extends `FloatingStack`); `window.toggle_floating` on a stacked member = ungroup (leave stack, float at frame rect + cascade offset).
- `window.open` with `StackTargetWindowID` joins the target's frame (creating it when the target is solo) with the new window active. ID reservation: IDs are unique across live windows ∪ all windows retained in `ClosedEntries`; a caller-supplied ID colliding with either → `ErrInvalidCommand`; with empty `ID` the daemon generates `w-<26-char-random>` (regenerating on reserved collision) and returns it in `Result`.

### API Endpoints

**No route additions or removals.** The existing surface carries everything:

- `POST /api/workspaces/:id/window-manager/commands` + `/preview` accept the 5 new command IDs and extended payloads (strict decode; unknown fields still rejected; `navigate` with `mode:"pop"` rejects a non-empty `route`; `close` validates `scope` against the closed set and forbids `minimize` with a non-empty scope).
- `GET /api/workspaces/:id/window-manager` returns the v3 wire snapshot (no `history`, plus `closed_entry_count`, `floating_stacks`, `nav_stack`, `pinned`).
- `GET .../window-manager/stream` frames are shape-compatible; client frames now carry `stack_active`.
- Layout endpoints validate/round-trip v3 documents; v2 documents fail with the existing `window_manager_invalid_topology` + `topology.unsupported_version` diagnostic (no converter).
- UDS mirrors byte-for-byte (existing parity harness extends to the new commands).

New deterministic error codes (contract + core mapping, HTTP 422):

```
window_manager_not_stacked      -- stack operation on a window not in any stack
window_manager_window_pinned    -- close refused: window is pinned (unpin first)
```

### Safety Invariants (marker 6)

1. A window referenced by any stack (node or `FloatingStack`) has `Placement == WindowPlacementStacked`; every window appears in exactly one container (group tree, floating list, or floating stack) — membership-exactly-once extends to floating stacks.
2. Every stack (both kinds) has ≥2 members and a member `ActiveID`; normalize dissolves 1-member stacks (node → leaf/tiled; floating stack → floating window at frame rect) and resets invalid `ActiveID` to the first member.
3. Pinned members form a contiguous prefix of `WindowIDs`; `window.stack.reorder` clamps within the region; normalize re-collates when violated.
4. All durable mutations go through `Manager.Execute` under CAS (`ExpectedRevision` + rebase guard). The active-tab coalescer flushes synchronously at the top of every durable `Execute` for the workspace via `flushStackActiveLocked` (caller holds the per-workspace lock); a failed pre-durable flush **fails the command** — structural commands never observe stale `ActiveID`. Timer-path flushes run on the Manager's detached context, take the workspace lock themselves, and retain pending values on failure. `Close()` cancels timers, flushes each workspace, and joins the timer goroutines before returning.
5. `window.stack.set_active` and coalescer flush commits are history-exempt: they never clear Redo and never create history entries. They DO emit the `window_manager.stack.activated` hook and a normal event frame; raw per-client focus flips emit neither.
6. `window.navigate` (all modes) stays history-exempt. `NavStack` growth is capped at write time by the effective workspace `nav_stack_limit` (oldest evicted in the reducer); `ValidateSnapshot` enforces only the absolute maximum (200). Undo/redo preserves live `Route` **and** `NavStack` (`restoreStatePreservingRoutes` extended).
7. `ClosedEntries` growth is capped at write time by the effective workspace `closed_entry_limit` (oldest evicted in the reducer); `ValidateSnapshot` enforces only the absolute maximum (100). Entries are pushed only by non-minimize closes and consumed only by `window.reopen`. Lowering a live limit is non-retroactive: it takes effect on the next relevant mutation.
8. `window.close` scope `""` on a pinned window fails with `window_manager_window_pinned`; scope `"group"` closes pinned members too (the sanctioned bulk path); scopes `"others"`/`"right"` never close pinned members. Every scope is one command = one `ClosedEntry`.
9. Window IDs are unique while **live or reopenable**: reservation spans live windows ∪ windows retained in `ClosedEntries`. Opens (caller-supplied or generated) never collide with either; evicting or consuming an entry releases its IDs. Reopened windows keep their original IDs.
10. `ClientView.StackActive` entries always reference live stacks and member windows; `repairClientView` prunes stale entries and seeds missing ones from durable `ActiveID`.
11. One user-level bulk operation = one command = one revision bump = one event frame = one hook batch = at most one `ClosedEntry` (`window.stack.group` for merge-all; close scopes for frame/others/right).
12. The wire snapshot never carries `History` or `ClosedEntries` bodies (count only); `ExportLayout` carries floating stacks, nav stacks, and pins but never closed entries.
13. Snapshot load discards and reinitializes ONLY on: version ≠ 3, or malformed/unknown-field decode failure. Workspace-ID mismatch and current-version topology validation failures remain hard errors that preserve the stored blob for forensics (never silently discarded).
14. All tab state is workspace-scoped inside the existing snapshot; no cross-workspace read/stream/cache path is added (workspace isolation preserved by construction).
15. History and `ClosedEntries` compose by exclusion: `ClosedEntries` is NOT part of the history `State` and undo/redo never mutate it. Undoing a close revives the window and leaves its entry stale (reopen skips live IDs per the consume rule); undoing a reopen removes the revived windows while their consumed entry stays consumed (bounded, documented loss). Close and reopen themselves remain history-eligible.

### Web Design

- **Frame-based group rendering** replaces per-window `display:none` stacking: `os-win-layer` renders one `<Rnd>` (floating) or one positioned frame (tiled pane) per group/stack, containing the deck (≥2 members), the active member's head/strip/body, and inactive members' subtrees kept mounted but hidden inside the same frame (state preservation, PRD US-003 AC-3). `disableDragging`/`display:none` per-member logic and `preferredActiveWindow` are deleted.
- **Deck component** (`systems/os/components/os-window-deck.tsx` + `os-window-tab.tsx`): 37px row per the reference — traffic lights (frame close = `CloseWindowCommand.Scope = "group"`), tab segments (glyph/state-dot + leaf label, close ×, attention badge, pinned form), `+` button (⌘T → new-tab), drag region. Tab click → `focusWindow` (activation = focus, ADR-009); middle-click/× → close; ⌥-click × → close others; drag within deck → `window.stack.reorder`; drag out → `window.toggle_floating` with pointer rect; context menu per PRD (close/close others/close right/pin/move to new window/merge all).
- **Semantic lookup** (`ADR-010`): `osWindowId` deleted; launch/focus/deep-link resolution scans `windows` by `(app, instanceKey)` with MRU ordering from `FocusOrder`; dock aggregates per app and cycles instances; sessions dedup by `instanceKey === sessionId`.
- **Routing coordinator**: causes unchanged; window resolution swaps to semantic lookup; `route-pop` navigations classify as `replace`, in-window drill-in links as `push`, breadcrumb/⌘[ as `pop` — classification happens where navigation intent is known (link helpers + topbar back), not inferred from paths.
- **Shortcuts/palette**: registry gains `window.tab.new|next|previous|last|reopen` + `window.tab.jump.1..8` (TS union + Go allow-list stay mirrored); ⌘W keeps `window.close` (the focused window *is* the active tab). Palette gains the "Go to tab" group (all windows across desktops, attention state shown, select = switch desktop + focus) and tab actions; new-tab picker is the palette scoped to destinations.
- **New-tab pseudo-app**: `OsAppId "new-tab"` registry entry (`dock: null`, `paths: ["/new-tab"]`, empty-state controller that opens the scoped picker) + a stub route so a focused empty tab owns a real URL.
- **D3 strip migration**: Topbar `nav` slot deleted from `TopbarSlotValue` and the Topbar zone; `tasks-catalog-location` and `marketplace-kind-page` publish views as the toolbar's leading pill-group (strip order: views · filters · spacer · display-mode).
- **Context-menu primitive**: `packages/ui` `context-menu.tsx` finalized first (exported from `index.ts`, story, test) — prerequisite for launch-surface and tab menus.

## Integration Points

None outside the codebase — no external services, auth systems, or third-party APIs are involved. (Section retained to state that explicitly.)

## Impact Analysis

| Component | Impact Type | Description and Risk | Required Action |
| --- | --- | --- | --- |
| `internal/windowmanager` types/validate/normalize | Modified | New fields + invariants; medium risk (canonical suites guard) | Extend topology/lifecycle suites |
| Reducers + history | Modified | 5 new commands, 3 extended payloads, ungroup semantics; medium-high | Extend lifecycle/layout suites |
| Manager clients/coalescer | New logic | Per-client StackActive + flush ordering; concurrency-sensitive | New unit + race coverage |
| `internal/daemon` repository | Modified | v3 discard-on-load replaces brick; low | Repository suite: discard cases |
| `internal/api/contract` + core | Modified | Wire deltas, History removal, 2 error codes; breaking wire change (sanctioned) | `make codegen` + contract tests |
| OpenAPI/TS generated types | Regenerated | Snapshot/window/client shapes change | `make codegen`, web adopts |
| `internal/hooks` | Modified | 4 events + ChangeSet outcome fields; low | Dispatch/introspection tests |
| `internal/cli` + generated docs | Modified | 5 new verbs + list output; low | CLI tests + `make codegen` |
| `internal/config` | Modified | 2 keys + shortcut allow-list additions; low | Config suite + docs |
| Native tools | Modified | 26→31 tools, toolset unchanged; low | Native-tool tests |
| `web/systems/os` rendering | Refactor | Frame-based groups; highest-risk area (gestures, projection, mounting) | Runtime/projection/component suites + e2e |
| `web` identity/routing/dock/palette | Refactor | Semantic lookup everywhere `osWindowId` lived; high | Coordinator/runtime/dock suites + e2e |
| `packages/ui` Topbar + context-menu | Modified | `nav` slot deleted; context-menu finalized; low | Stories + tests |
| `web` tasks/marketplace routes | Modified | Views → strip (D3); low | Route component tests + screenshot |
| `skills/compozy/references/window-management.md` | Modified | New commands/verbs/tools/config documented | Update with feature |
| `docs/qa/scenarios/ET-window-manager-*` + web lifecycle | Reset/extend | Behavior changes reset affected scenarios; new tab scenarios added | qa-report task |

## Testing Approach

Strategy only — cases live in `_tests.md`. Go: table-driven `t.Run("Should …")` + `t.Parallel`, race-enabled, fakes only at the clientstate boundary; extend the canonical in-package suites (`topology_test.go`, `lifecycle_test.go`, `layout_commands_test.go`, `service_test.go`, `service_contract_test.go`, `config_test.go`) rather than new files, plus daemon repository/native-tool, hooks dispatch, api core/ws, and CLI parity suites. Web: Vitest + Testing Library in the existing os-system suites (runtime, routing-coordinator, store, projection, dock/palette/window components) with mocked transport only; visual states via Storybook stories + `eng-ui-screenshot` captures. E2E: extend `web/e2e/__tests__/os-shell.spec.ts` (browser, real daemon) with deck journeys; CLI/HTTP/UDS parity via the existing transport-parity integration harness. QA: scenario files reset/added per the QA tracker rule, executed by the trailing qa-execution task.

## Development Sequencing

### Build Order

1. **Domain foundation** — types (FloatingStack, ClosedEntry, NavStack, Pinned, StackActive), validate + normalize extensions, v3 constant. No dependencies.
2. **Reducers** — group/reorder/set_active/pin/reopen, close-group + pinned refusal + closed-entry push, navigate modes, move-center-to-floating, toggle_floating ungroup, open stack-target + ID generation. Depends on 1.
3. **Manager** — ClientView.StackActive + repair, activeCoalescer + flush ordering, history/hook exemptions. Depends on 2.
4. **Persistence + config** — repository v3 discard; `nav_stack_limit`/`closed_entry_limit` + shortcut allow-list. Depends on 1.
5. **Contract + codegen** — wire types, History removal, error codes, conversions; `make codegen` (OpenAPI + TS + CLI docs). Depends on 2-4.
6. **Hooks + native tools + CLI** — 4 events + ChangeSet outcomes; 5 tools; 5 verbs + list output; docs regen. Depends on 5.
7. **Web foundation** — generated types adoption, semantic lookup (delete `osWindowId`), tab command layer in new `runtime/window-manager-tab-commands.ts` (group/reorder/activate/pin/reopen/close scopes/navigate modes), frame-based rendering + group projection in new `lib/group-projection.ts`. Depends on 5.
8. **Deck UI + menus** — context-menu primitive finalization (packages/ui, first), deck/tab components, tab context menus, launch-surface destination menus, drag out/in/reorder gestures. Depends on 7.
9. **Shortcuts + palette + new-tab** — registry/allow-list mirror, tab actions, Go-to-tab group, new-tab app + stub route. Depends on 7.
10. **D3 strip migration** — Topbar `nav` slot removal, tasks/marketplace strip views, DS document amendments (os-shell.html §02/§07, pagehead-redesign.html §05). Depends on 8 (deck defines strip conventions).
11. **Docs + skill** — `skills/compozy/references/window-management.md`, config-toml.mdx reference rows, lifecycle-matrix. Depends on 6.
12. **QA pair** — qa-report scenario planning + qa-execution (per `cy-tasks-tail-qa-pair`).

### Technical Dependencies

No external blockers. Internal prerequisite: the `packages/ui` context-menu primitive must be exported/storied/tested before any menu work (step 8).

## Monitoring and Observability

- Hook events are the operational signal: `window_manager.window.opened/closed`, `window_manager.stack.grouped/ungrouped/activated`, plus existing `layout.applied`/`desktop.*`/`window.moved` — all carrying workspace, revision, command, ChangeSet, actor, origin.
- Structured logs: v3 discard warning (`workspace_id`, stored version, reason) before row deletion; coalescer flush failures (warn, non-fatal); slow-consumer evictions unchanged.
- No new metrics endpoints; `compozy layout watch` streams the same frames for live observation.

## Technical Considerations

### Key Decisions

Recorded as ADRs (below); headline trade-offs: reuse-over-invent for topology (ADR-008), presentation-first activation accepting a bounded last-active lag in exchange for per-client semantics and quiet wires (ADR-009), semantic lookup accepting O(windows) scans in exchange for coordination-free IDs (ADR-010), explicit navigate modes over route-hierarchy inference (ADR-011), one-time state loss over migration code (ADR-012), and daemon-side reopen storage over client convenience (ADR-013).

### Known Risks

- **Frame-based rendering refactor** touches gestures, projection, mounting, and the 12-window e2e envelope — the highest-risk slice; mitigated by keeping member subtrees mounted (no lifecycle change for apps) and extending the projection/gesture suites before UI work.
- **`ReturnAnchor` fidelity**: stored `SourceGroup` clones predate grouping changes and byte-compare on restore; anchors created before a grouping op degrade to fallback insertion — accepted (documented behavior, v3 reset clears old anchors).
- **Semantic-lookup completeness**: any missed `osWindowId` caller reintroduces singletons — mitigated by deleting the symbol (compile break) and launch-surface test coverage.
- **Coalescer correctness under shutdown/concurrency** — the Manager has two lock layers (registry + per-workspace); the coalescer's contract (workspace-lock binding, `flushStackActiveLocked` split, detached timer context, Close cancel+flush+join, fail-the-command on pre-durable flush failure) is specified in Core Interfaces and covered by dedicated race tests (timer vs Execute, unregister vs timer, Close vs timer, failed-flush retry, multi-workspace independence).

## Delete Targets

Per L-006 — removed in the same change that replaces each:

- `SnapshotVersion = 2` acceptance everywhere; the load-error **brick path** for version mismatch (replaced by discard-and-reinit).
- Wire `WindowManagerSnapshot.History` field + its conversions + web schema field (History becomes domain-internal).
- `osWindowId()` and the `app:<app>` / `session:<key>` ID scheme (`os-types.ts:189-191`) + every derivation call site.
- Dock instance-key skip (`use-desktop-dock.ts:44`).
- `preferredActiveWindow` focus-override (`layout-projection.ts:167-172`).
- Per-member stack hiding + drag-disable in `os-window.tsx` (`display:none` at :84, `disableDragging` at :73) — replaced by frame-based rendering.
- Topbar `nav` slot (`TopbarSlotValue.nav`, the `topbar-nav` zone) + both head-hosted views call sites (moved to strip).
- S2/S3 rule text in `os-shell.html` §07 and the route-nav row of `pagehead-redesign.html` §05 (amended per ADR-007).
- v2 `window_layout` documents: no converter — rejected with the existing deterministic diagnostics.

## Extensibility Integration Plan

- **Hooks**: +5 typed events (opened/closed/grouped/ungrouped/**activated**) with payloads, dispatch mapping, and introspection entries; existing 4 events unchanged. `stack.activated` fires on durable activation commits (explicit `set_active` and coalesced flushes — the durable write is the observable record, PRD US-027 EC-1); raw per-client focus flips emit no hook.
- **Native tools/resources**: +5 tools in toolset `compozy__window_manager` (group/reorder/activate/pin/reopen) with descriptors, I/O schemas, toolmeta display entries, and availability gates matching the existing 26.
- **Skills**: `skills/compozy/references/window-management.md` updated (commands, verbs, tools, config keys, tab model).
- **Layout resources**: `window_layout` documents extend to v3 shapes; registry/codec validation covers floating stacks/nav/pins.
- **No impact** (checked): extension manifests and the provide/permission surfaces (no new Host API methods), bundles, registries beyond layout resources, bridge SDKs, MCP sidecars, protocol docs (Compozy Network untouched).

## Agent Manageability Plan

- **Commands**: the 5 new IDs + 3 extended payloads on `POST .../commands` + `/preview`, HTTP/UDS parity asserted by the transport-parity harness.
- **CLI**: `compozy window group --target <id> --windows <a,b,…> [--insert-index N]`, `compozy window activate <id> [--client]`, `compozy window pin <id> --pinned=<bool>` (aliases `pin`/`unpin`), `compozy window reopen`, extended `compozy window list` (stack id, members order, active, pinned, nav depth) — plus the extended existing verbs shipping in the same pass: `compozy window open --stack-target <id>`, `compozy window navigate --mode replace|push|pop` (route flags forbidden with `pop`), `compozy window close --scope tab|group|others|right`. All with `-o human|json|jsonl|toon`; docs regenerated via `make codegen`.
- **Native tools**: new `compozy__window_{group,reorder,activate,pin,reopen}` plus schema updates to existing `compozy__window_{open,navigate,close}` (stack target / mode / scope) — descriptors, I/O schemas, and digests co-ship.
- **Naming**: the public noun is **activate** everywhere (CLI verb, native tool, hook `stack.activated`); `window.stack.set_active` is the internal command ID — the equivalence is documented once in the skill reference and CLI docs.
- **Discovery**: snapshot (`GET`/`compozy layout get`) exposes full tab topology + `closed_entry_count`; `compozy layout watch` streams changes; settings surface exposes the new config keys.
- **Deterministic errors**: `window_manager_not_stacked`, `window_manager_window_pinned`, plus existing codes; documented per verb in generated CLI docs and the skill reference.

## Config Lifecycle

- **New keys** under `[window_manager]`: `nav_stack_limit int` (default 50, range 1..200), `closed_entry_limit int` (default 20, range 1..100) — struct fields, defaults, validation, workspace overrides (same pointer-override pattern as existing keys), settings GET/PATCH exposure, `config-toml.mdx` sample + reference rows, `lifecycle-matrix.mdx` rows (`live`/`live`/`none`), config suite coverage. **Enforcement semantics**: both are reducer write-time caps resolved from the effective workspace config at mutation time; `ValidateSnapshot`/`NormalizeSnapshot` stay config-free and enforce only the absolute maxima (200/100); lowering a live value is non-retroactive (documented in the reference row).
- **Changed validation**: `CanonicalShortcuts` allow-list gains `window.tab.new|next|previous|last|reopen|jump.1..8` (Go) mirrored in the TS `WindowManagerActionId` union; conflict detection unchanged.
- **No removals**; all other `[window_manager]` keys untouched (checked: policies, gaps, snap, bindings, history_limit).

## Web/Docs Impact

- `web/`: `systems/os` (rendering, deck, identity, coordinator, dock, palette, shortcuts, new-tab app), `systems/marketplace` + `systems/os/apps/tasks` (strip views), `packages/ui` (Topbar, context-menu). Generated TS types refresh via `make codegen`.
- `packages/site`: regenerated CLI docs (new verbs), `configuration/config-toml.mdx`, `configuration/lifecycle-matrix.mdx`.
- Design docs: `docs/design/opendesign/design-system/os-shell.html` §02/§07 + `os/pagehead-redesign.html` §05 amended; `window-tabs-variations.html` stays the reference.
- QA: reset `ET-window-manager-{layout-gestures,drop-swap,multi-client,public-parity,layout-recovery,hooks-resources}`, `ET-web-window-routing-lifecycle`, `RT-home-usage-window-persistence`, `MS-configure-window-manager`, `MS-layout-profile-cli-roundtrip`; add tab scenarios (deck, reopen, pins, multi-instance, tab search, agent tab management).

## Assumptions / Defaults

- Deck renders only at ≥2 members (domain invariant); compact/small-screen (<960px) keeps current behavior — group active member fills the frame, no deck (PRD open question stays open; nothing here depends on it).
- Generated window ID format `w-<random>` is opaque; no consumer may parse IDs (enforced by review, not runtime).
- Last-active coalescer quiet period is 2s (internal constant, not config — no known need to tune).
- Reopen restores into the current desktop when the original is gone; no cross-workspace reopen.
- `InstanceKey` remains a passthrough with client-defined semantics (sessions: session id); the daemon still never enforces uniqueness.
- Deep-link instance targeting = most-recently-focused instance of the resolved app (PRD BR-11), using client `FocusOrder`.
- Session windows inside groups keep their existing cross-workspace guard behavior (close + reopen agents window).
- E2E keeps the 12-window envelope as the perf ceiling; tab groups don't raise it in v1.

## Architecture Decision Records

- [ADR-001: Window tabs are grouped windows](adrs/adr-001.md) — product model: tab = full window; one concept.
- [ADR-002: Apps become multi-instance](adrs/adr-002.md) — N instances per app; focus-first launch.
- [ADR-003: Tab persistence split](adrs/adr-003.md) — durable membership; per-client active.
- [ADR-004: Frictionless close + reopen](adrs/adr-004.md) — no confirmations; ⌘⇧T multi-level.
- [ADR-005: Windows by default; tabs explicit](adrs/adr-005.md) — context-menu destinations; no global preference.
- [ADR-006: v1 extras](adrs/adr-006.md) — tab search, menu merge/move, pins in; peek out.
- [ADR-007: Route views to the strip everywhere](adrs/adr-007.md) — D3/D4 amendment, global.
- [ADR-008: Stack node + FloatingStack representation](adrs/adr-008.md) — reuse tiled stacks; floating sibling container.
- [ADR-009: Focus-driven activation + coalesced last-active; History off wire](adrs/adr-009.md) — per-client active mechanics.
- [ADR-010: Generated window IDs + semantic lookup](adrs/adr-010.md) — multi-instance identity.
- [ADR-011: Durable nav stacks via navigate modes](adrs/adr-011.md) — push/pop/replace.
- [ADR-012: Snapshot v3 hard cut with boot discard](adrs/adr-012.md) — no upgrade path.
- [ADR-013: Daemon-durable reopen; frame close = one entry](adrs/adr-013.md) — ClosedEntries + window.reopen.
