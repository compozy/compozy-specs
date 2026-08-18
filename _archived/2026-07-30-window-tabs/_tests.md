# Test Specification: Window Tabs (the Deck)

Canonical test contract for window tabs. Companion to `_techspec.md`. Derived from `_user_stories.md` (behavior) and `_techspec.md` (components). IDs are local to this contract; implementation extends the existing canonical suites (Go in-package suites, web os-system suites, `web/e2e/__tests__/os-shell.spec.ts`) rather than creating parallel files — the physical e2e suite continues its own numbering.

## Strategy

- **Go**: table-driven `t.Run("Should …")` subtests, `t.Parallel` default, `-race`/`CGO_ENABLED=1`; fakes only at the clientstate boundary (existing memory adapters). Extend: `topology_test.go`, `lifecycle_test.go`, `layout_commands_test.go`, `service_test.go`, `service_contract_test.go`, `config_test.go`, daemon `window_manager_repository_test.go` + `native_tools_test.go`, `hooks/window_manager_dispatch_test.go`, `api/core/window_manager{,_ws}_test.go`, `cli/window_manager_test.go`, `api/udsapi/transport_parity_integration_test.go`, `api/contract/contract_test.go`.
- **Web**: Vitest + Testing Library (jsdom) in the existing `web/src/systems/os` suites; transport mocked at the adapter boundary only; stories for deck visual states.
- **E2E**: Playwright `os-shell.spec.ts` against a real daemon (`make test-e2e-web`); CLI journeys via the CLI/UDS harness.
- Status-code **and** body assertions on every API case; deterministic error codes asserted by exact string.

## Coverage Matrix

| Source | Behavior | Unit | Integration | E2E |
| --- | --- | --- | --- | --- |
| US-001 | Group two windows into one frame | UT-010, UT-011, UT-012 | IT-003 | E2E-001 |
| US-001.EC-1 | Merging a tabbed window folds all tabs in order | UT-013 | — | — |
| US-001.EC-2 | Cross-desktop group lands on target desktop | UT-014 | — | — |
| US-001.EC-3 | Concurrent groups serialize via CAS | UT-060 | IT-004 | — |
| US-001.EC-4 | Self-group → reorder, nothing new | UT-015 | — | — |
| US-002 | Deck at ≥2, gone at 1; controls migrate | UT-070, UT-071 | — | E2E-002 |
| US-002.EC-1 | Rapid add/remove keeps chrome consistent | UT-071 | — | E2E-002 |
| US-002.EC-2 | Sole pinned survivor: deck unmounts, pin retained | UT-016, UT-072 | — | — |
| US-003 | Head/strip/content follow active tab | UT-073 | — | E2E-003 |
| US-003.EC-1 | Loading tab activates instantly | UT-074 | — | — |
| US-003.EC-2 | Tab switch cancels in-flight gesture safely | UT-075 | — | — |
| US-003.EC-3 | Keyboard activation beats hover | UT-076 | — | — |
| US-004 | Truthful labels, min width, scroll overflow | UT-077, UT-078 | — | E2E-004 |
| US-004.EC-1 | Hostile/long/RTL titles render safely + tooltip | UT-078 | — | — |
| US-004.EC-2 | Untitled empty tab shows "New tab" | UT-079 | — | — |
| US-004.EC-3 | 30+ tabs: scroll + all reachable | UT-077 | — | E2E-013 |
| US-005 | ⌘T empty tab + destination picker | UT-080, UT-024 | — | E2E-005 |
| US-005.EC-1 | Repeated ⌘T stacks empty tabs; each closable | UT-080 | — | — |
| US-005.EC-2 | Dismissed picker keeps "New tab" | UT-081 | — | — |
| US-005.EC-3 | ⌘T with no focused window opens a window | UT-082 | — | — |
| US-006 | Launch-surface context menu destinations | UT-083, UT-084 | — | E2E-006 |
| US-006.EC-1 | "Go to tab" for already-open item | UT-084 | — | — |
| US-006.EC-2 | Menu inert/closed under overlays | UT-085 | — | — |
| US-006.EC-3 | Keyboard-only destination choice | UT-086 | — | E2E-006 |
| US-007 | Close instant, silent, session survives | UT-020, UT-021 | IT-005 | E2E-007 |
| US-007.EC-1 | Closing last tab of only window empties desktop | UT-022 | — | — |
| US-007.EC-2 | Concurrent double close = one close + no-op | UT-060 | IT-004 | — |
| US-007.EC-3 | Agent closes viewed tab; client follows live | — | IT-006 | E2E-011 |
| US-007.EC-4 | Close mid-drag cancels gesture cleanly | UT-075 | — | — |
| US-008 | ⌘⇧T restores newest-first with nav stack | UT-025, UT-026 | IT-005 | E2E-007 |
| US-008.EC-1 | Reopen on empty history = quiet no-op | UT-027 | — | — |
| US-008.EC-2 | Reopen of deleted session lands on truthful state | UT-087 | — | — |
| US-008.EC-3 | Original group gone → standalone window | UT-026 | — | — |
| US-008.EC-4 | Reopen history survives client reload | — | IT-005 | E2E-007 |
| US-009 | Pin: left collate, glyph-only, refuse close | UT-016, UT-017, UT-023 | — | E2E-008 |
| US-009.EC-1 | All-pinned frame: ⌘W inert, frame close works | UT-023, UT-028 | — | — |
| US-009.EC-2 | Unpin restores label/close at region boundary | UT-017 | — | — |
| US-009.EC-3 | Dragged-out pinned window keeps flag | UT-018 | — | — |
| US-010 | Close others / close right / ⌥-click | UT-088, UT-172 | — | E2E-008 |
| US-010.EC-1 | Close others vs all-pinned = disabled no-op | UT-088 | — | — |
| US-010.EC-2 | Bulk close keeps sessions + dock signals | — | IT-006 | E2E-011 |
| US-011 | Per-tab nav stack; ⌘[ pops only active | UT-030, UT-031 | — | E2E-009 |
| US-011.EC-1 | Pop at root = no-op success | UT-032 | — | — |
| US-011.EC-2 | Deleted entity shows gone-state on activation | UT-089 | — | — |
| US-011.EC-3 | Go-to-tab arrives at current depth | UT-090 | — | E2E-014 |
| US-012 | Labels derive from leaf, live | UT-091 | — | E2E-003 |
| US-012.EC-1 | Colliding leaves disambiguated by tooltip path | UT-078 | — | — |
| US-012.EC-2 | No rename affordance exists | UT-092 | — | — |
| US-013 | Drag tab out → window at pointer, stack intact | UT-033, UT-093 | — | E2E-010 |
| US-013.EC-1 | Dragging active tab refocuses neighbor + new window | UT-093 | — | — |
| US-013.EC-2 | Drag out of tiled pane keeps group valid | UT-033 | — | E2E-010 |
| US-013.EC-3 | Interrupted drag leaves no half state | UT-094 | IT-004 | — |
| US-014 | Drop affordance before merge; cancelable | UT-095, UT-179 | — | E2E-001 |
| US-014.EC-1 | Cross-desktop drop rules | UT-014 | — | — |
| US-014.EC-2 | Scrolled deck shows insertion point + edge scroll | UT-095 | — | — |
| US-014.EC-3 | Re-merged pinned window joins pinned region | UT-016 | — | — |
| US-015 | Merge-all / move-to-window via menu | UT-012, UT-096 | IT-003 | E2E-012 |
| US-015.EC-1 | Merge-all with one window = disabled no-op | UT-096 | — | — |
| US-015.EC-2 | Minimized windows join unminimized | UT-012 | — | — |
| US-015.EC-3 | Pins survive merge into pinned region | UT-016 | — | — |
| US-016 | Second instance with independent state | UT-040, UT-041 | — | E2E-013 |
| US-016.EC-1 | No instance cap; scroll + search escape | UT-077 | — | E2E-013 |
| US-016.EC-2 | Closing one instance never touches the other | UT-041 | — | — |
| US-016.EC-3 | Deep link lands on MRU instance | UT-042 | — | E2E-014 |
| US-017 | Focus-first, cycle on repeat | UT-043, UT-044 | — | E2E-013 |
| US-017.EC-1 | One activation reaches tab on other desktop | UT-044 | — | E2E-014 |
| US-017.EC-2 | Cycle un-minimizes in turn | UT-043 | — | — |
| US-018 | Live state dots/badges on tabs, accent discipline | UT-097 | — | E2E-011 |
| US-018.EC-1 | Compressed/scrolled tab keeps badge; dock aggregates | UT-097 | — | — |
| US-018.EC-2 | Independent badges, no aggregate wash | UT-097 | — | — |
| US-018.EC-3 | State flapping renders latest without storms | UT-098 | — | — |
| US-019 | Signals outlive tabs via dock/sessions | UT-045 | IT-006 | E2E-011 |
| US-019.EC-1 | Needs-input with no tab still routes from dock | UT-045 | — | E2E-011 |
| US-019.EC-2 | Answered signal clears everywhere live | — | IT-006 | — |
| US-020 | Full keyboard contract + rebinding | UT-050, UT-051, UT-099 | — | E2E-005 |
| US-020.EC-1 | Editable targets keep keys | UT-099 | — | — |
| US-020.EC-2 | ⌘1–8 positional, ⌘9 last, rest reachable | UT-050 | — | E2E-005 |
| US-020.EC-3 | Rebound chord honored, no hardcoded fallback | UT-051 | — | — |
| US-021 | Palette "Go to tab" across desktops | UT-100, UT-101 | — | E2E-014 |
| US-021.EC-1 | Identical labels disambiguated by context | UT-100 | — | — |
| US-021.EC-2 | Minimized target un-minimizes on jump | UT-101 | — | — |
| US-021.EC-3 | No matches → group absent, no dead end | UT-100 | — | — |
| US-022 | Full restore: groups/order/pins/nav/last-active | UT-034, UT-061 | IT-001, IT-002 | E2E-015 |
| US-022.EC-1 | Unresolvable surface restores to truthful state | UT-087 | — | — |
| US-022.EC-2 | Pre-tabs snapshot: discard-and-reinit at load | UT-062 | IT-002 | — |
| US-022.EC-3 | 100× volume stays responsive; decks scroll | UT-077, UT-178 | — | E2E-013 |
| US-023 | Per-client active over shared membership | UT-063, UT-064 | IT-007 | E2E-016 |
| US-023.EC-1 | Peer close of my active tab follows policy live | UT-064 | IT-007 | — |
| US-023.EC-2 | Rejoining client re-derives from last-active | UT-063 | IT-007 | — |
| US-024 | Tiled pane decks; stack switcher fixed | UT-035, UT-102 | — | E2E-002 |
| US-024.EC-1 | Last non-active drag-out → normal tile | UT-033 | — | — |
| US-024.EC-2 | Narrow pane compresses/scrolls deck | UT-102 | — | — |
| US-024.EC-3 | Undo/redo restores membership consistently | UT-036 | — | — |
| US-025 | Structured topology inspection | UT-110, UT-111 | IT-008 | E2E-017 |
| US-025.EC-1 | Zero groups: fields present, empty | UT-110 | — | — |
| US-025.EC-2 | New shape only; no dual output | UT-112 | IT-009 | — |
| US-026 | Agent tab mutations, parity, revision-safe | UT-113, UT-114 | IT-008, IT-010 | E2E-017 |
| US-026.EC-1 | Concurrent agent+human serialize; stale gets conflict | UT-060 | IT-004 | — |
| US-026.EC-2 | Agent closes/moves operator's active tab live | — | IT-006 | E2E-011 |
| US-026.EC-3 | Command on just-closed tab → window_not_found | UT-114 | — | — |
| US-026.EC-4 | Malformed group specs → specific validation errors | UT-011 | IT-010 | — |
| US-027 | Tab lifecycle hook events | UT-120, UT-121 | IT-011 | — |
| US-027.EC-1 | Per-client activation emits no durable event | UT-122 | IT-011 | — |
| US-027.EC-2 | Bulk op = one coherent batch per revision | UT-121 | IT-011 | — |
| US-028 | Layout export/apply/profiles round-trip groups | UT-037, UT-038 | IT-012 | E2E-017 |
| US-028.EC-1 | Pre-tabs document rejected with version diagnostic | UT-039 | IT-012 | — |
| US-028.EC-2 | Unresolvable surfaces follow standing apply rules | UT-038 | — | — |
| US-029 | Route views in strip everywhere | UT-130, UT-131 | — | E2E-018 |
| US-029.EC-1 | Views-only strip renders alone | UT-130 | — | — |
| US-029.EC-2 | No peer views → no views group | UT-131 | — | — |
| Domain validate/normalize (TechSpec §Data Models) | v3 invariants 1-3, 10 | UT-001–UT-009 | — | — |
| ID reservation + history×ClosedEntries (invariants 9, 15) | reopen/undo composition | UT-173, UT-174, UT-175 | — | — |
| Manager/coalescer (TechSpec §Core) | invariants 4-6 | UT-060–UT-066, UT-176 | IT-007 | — |
| Repository (ADR-012) | invariant 13 | UT-062 | IT-002 | — |
| Contract/wire (TechSpec §API) | shapes + codes + invariant 12 | UT-112, UT-115 | IT-009 | — |
| Config lifecycle | 2 keys + allow-list | UT-140–UT-143 | — | — |
| CLI verbs | 5 new + list | UT-150–UT-154 | IT-010 | E2E-017 |
| Native tools | 5 new bindings | UT-160 | IT-008 | — |
| context-menu primitive (packages/ui) | export + behavior | UT-170 | — | — |

## Unit Tests

### Domain: types, validate, normalize (`topology_test.go`)

- **UT-001** (happy): `ValidateSnapshot` accepts a v3 snapshot with a 2-member `FloatingStack` whose members have `Placement=stacked` and `ActiveID` a member.
- **UT-002** (error): `FloatingStack` with 1 member → diagnostic `topology.stack_size`; `ActiveID` non-member → `topology.stack_active`; member with `Placement=floating` → `topology.window_placement`.
- **UT-003** (error): window referenced by both a `FloatingStack` and `Desktop.Floating` → `topology.window_membership`.
- **UT-004** (state): `NormalizeSnapshot` dissolves a 1-member `FloatingStack` into a floating window at the stack's `Rect` and drops the stack.
- **UT-005** (state): normalize resets `FloatingStack.ActiveID` to first member when stale; same for node stacks (regression guard).
- **UT-006** (state): normalize re-collates pinned members into a contiguous prefix preserving relative order (node stack + floating stack).
- **UT-007** (boundary): `Snapshot.Version != 3` → `topology.unsupported_version`; v3 accepted.
- **UT-008** (boundary): `window.navigate{mode:push}` with the effective workspace `nav_stack_limit` reached evicts the oldest entry at write time (reducer test with explicit effective config); `ValidateSnapshot` accepts stacks up to the absolute maximum (200) regardless of config and rejects beyond it; normalize never truncates by config.
- **UT-009** (boundary): `window.close` with the effective `closed_entry_limit` reached evicts the oldest `ClosedEntry` at write time (reducer test); validation enforces only the absolute maximum (100); lowering the live limit is non-retroactive (existing over-limit entries survive until the next close); entries referencing missing desktops are retained (fallback applies at reopen).

### Reducers: grouping (`lifecycle_test.go`, `layout_commands_test.go`)

- **UT-010** (happy): `window.stack.group` with floating target + one floating member creates a `FloatingStack{target, member}`, member active last-joined, both `Placement=stacked`.
- **UT-011** (error): group with unknown target → `ErrWindowNotFound`; member==target → `ErrInvalidCommand`; duplicate member ids → `ErrInvalidCommand`; empty `WindowIDs` → `ErrInvalidCommand`.
- **UT-012** (happy): group with 4 windows (one minimized) onto a tiled leaf builds a 5-member node stack, minimized member un-minimizes, join order preserved after existing members.
- **UT-013** (state): grouping a window that is itself a `FloatingStack` folds all its members in order into the target and dissolves the source stack.
- **UT-014** (state): grouping a member from desktop B onto a target in desktop A moves it to A (`DesktopID` rewritten, origin desktop no longer references it).
- **UT-015** (state): `window.move` `DropCenter` where source already belongs to the target stack reorders instead of duplicating (window appears exactly once).
- **UT-016** (state): pinned windows joining a stack collate into the pinned prefix; merge of two stacks interleaves pinned-first then unpinned, relative order stable.
- **UT-017** (state): `window.pin{pinned:false}` moves the window to the head of the unpinned region; label/close behavior flags cleared (wire `pinned=false`).
- **UT-018** (state): `window.toggle_floating` on a pinned stacked member ungroups it; `Pinned` flag survives on the standalone window.
- **UT-019** (state): `window.stack.reorder` moves an unpinned member to `Index` within the unpinned region; index into the pinned region clamps to the boundary; not-stacked window → `ErrNotStacked`.

### Reducers: close + reopen (`lifecycle_test.go`)

- **UT-020** (happy): `window.close` (non-minimize) on a 3-member stack removes the member, pushes one `ClosedEntry{1 window}` with route+navstack+pin, neighbor becomes `ActiveID` when the closed was active.
- **UT-021** (happy): `window.close{scope:"group"}` on any member closes all members (pinned included), pushes ONE `ClosedEntry{N windows, ActiveID, StackID, Rect}` preserving deck order; active-neighbor rule asserted on scope `""` (right neighbor first, else left).
- **UT-022** (state): closing the last tab of the workspace's only window leaves an empty desktop; snapshot validates.
- **UT-023** (error): `window.close{scope:""}` on a pinned window → `ErrWindowPinned` (code `window_manager_window_pinned`); `scope:"group"` succeeds; `minimize:true` with any non-empty scope → `ErrInvalidCommand`.
- **UT-024** (happy): `window.open` with `StackTargetWindowID` of a solo floating window creates a `FloatingStack{target, new}` with the new window active; on a solo tiled leaf creates a node stack.
- **UT-025** (happy): `window.reopen` pops the newest entry; a 1-window entry whose `StackID` still exists rejoins that stack as active; IDs/routes/navstacks/pins restored verbatim.
- **UT-026** (state): reopen of a frame entry whose stack is gone rebuilds a `FloatingStack` at `Rect` with recorded order + active; a 1-window entry with dead stack becomes a floating window.
- **UT-027** (boundary): reopen with empty `ClosedEntries` → `Result.Applied=false`, no error, revision unchanged.
- **UT-028** (state): frame close of an all-pinned frame succeeds and restores fully via reopen (pins intact).
- **UT-029** (idempotency): two sequential identical `window.reopen` calls restore two different entries (newest-first order asserted); reopen after history exhausted no-ops.
- **UT-172** (happy): `window.close{scope:"others"}` closes every unpinned member except the target as ONE multi-window `ClosedEntry` (pinned survive); `scope:"right"` closes strictly-after unpinned members in deck order; either scope with nothing to close → `Applied=false` success; one revision and one ChangeSet per call.
- **UT-173** (error): `window.open` with a caller-supplied ID matching a window retained in `ClosedEntries` → `ErrInvalidCommand` (reserved); generated IDs regenerate on reserved collision; consuming/evicting the entry releases the ID and the same open then succeeds.
- **UT-174** (state): close → `layout.undo` revives the window → `window.reopen` consumes the stale entry and skips the live ID (`Applied=false` when every entry window is live); a mixed entry restores only the dead members.
- **UT-175** (state): reopen → `layout.undo` removes the revived windows while the consumed entry stays consumed (bounded documented loss); `ClosedEntries` is bit-identical across the undo/redo cycle (invariant 15).
- **UT-179** (state): `window.stack.group` with `InsertIndex` splices joiners at the anchor (clamped to pin regions); nil appends; out-of-range clamps to the nearest legal position.

### Reducers: navigation (`lifecycle_test.go`)

- **UT-030** (happy): `window.navigate{mode:push}` appends prior `Route` to `NavStack` and sets the new route; `mode:pop` restores the last entry and shrinks the stack.
- **UT-031** (state): pushes beyond `nav_stack_limit` evict oldest; stack order preserved.
- **UT-032** (boundary): `mode:pop` with empty `NavStack` → `Applied=false` success, route unchanged.
- **UT-033** (state): drag-out (`window.toggle_floating` / `move floating`) preserves the window's `NavStack`; origin tiled group stays valid (no empty slot), 2→1 member stack collapses to leaf.
- **UT-034** (state): history undo of a structural change preserves live `Route` AND `NavStack` (`restoreStatePreservingRoutes` extension).
- **UT-035** (happy): `layout.arrange --arrangement stack` over 3 windows still builds a node stack (regression); deck-eligible (≥2) invariants hold.
- **UT-036** (state): undo after `window.stack.group` restores pre-group membership; redo re-applies it; `set_active` between them doesn't pollute history (history-exempt).
- **UT-037** (happy): `ExportLayout` v3 includes `floating_stacks`, `nav_stack`, `pinned`; excludes `closed_entries` and `history`.
- **UT-038** (happy): `layout.replace` with a v3 document containing floating stacks reconstructs them exactly; live routes re-grafted for surviving IDs.
- **UT-039** (error): `layout.replace` / profile apply with `document.Version=2` → `ErrInvalidTopology` with `topology.unsupported_version`.

### Web: identity + instances (`window-manager-runtime.test.ts`, `app-registry.test.ts`)

- **UT-040** (happy): "open new instance" of `tasks` while one exists dispatches `window.open` with a generated unique id (`w-` prefix) — no focus fallback.
- **UT-041** (state): two tasks instances hold independent `routeIntents`/view state; closing one leaves the other's state untouched.
- **UT-042** (happy): deep-link reconciliation picks the most-recently-focused instance whose app matches; no instance → opens one.
- **UT-043** (happy): dock click with instances [A,B] focuses MRU; repeated activation cycles A→B→A; minimized instance un-minimizes on its turn.
- **UT-044** (happy): palette/dock activation of an instance on another desktop issues desktop switch + focus + (stacked) activation in one user action.
- **UT-045** (state): closing the tab of a needs-input session keeps the dock sessions badge lit (attention model reads sessions, not windows).

### Web: shortcuts + registry (`window-manager-shortcuts.test.ts`, `use-os-shortcuts` suite)

- **UT-050** (happy): `window.tab.jump.3` activates the 3rd tab of the focused frame; `window.tab.last` the last; `window.tab.next/previous` cycle MRU-adjacent wrapping.
- **UT-051** (state): rebinding `window.tab.next` via config overrides dispatch; conflict detection flags a chord collision with `window.close`.
- **UT-052** (boundary): tab shortcuts in a single-tab window: jump/next no-op; `window.tab.new` mounts the deck via ⌘T path.

### Go: manager clients + coalescer (`service_test.go`)

- **UT-060** (concurrency): two concurrent `Execute` group commands on the same revision — one applies, the other gets `RevisionConflictError`; with valid rebase guard the stale one rebases.
- **UT-061** (happy): restart simulation (new manager over same store) restores groups/order/pins/navstacks; group's durable `ActiveID` seeds `StackActive` for a fresh client.
- **UT-062** (state): repository load of a v2 (or undecodable) snapshot deletes the row, reinitializes, and logs; subsequent `Execute` works from revision 1 (in daemon `window_manager_repository_test.go`).
- **UT-063** (happy): `window.focus` on a stacked member sets that client's `StackActive[stack]`; a second registered client's `StackActive` is untouched.
- **UT-064** (state): peer client closes my active tab → my `repairClientView` drops the stale `StackActive` entry and re-seeds; client frame emitted with bumped `PresentationRevision`.
- **UT-065** (concurrency): coalescer flush races a structural command — flush applies first under the mutex; `ActiveID` observed by the structural reduce is fresh; no deadlock under `-race`.
- **UT-066** (state): pending last-active flushes on `UnregisterClient` and on manager shutdown; flush commit creates no history entry and dispatches exactly one `window_manager.stack.activated`.
- **UT-176** (concurrency): coalescer lifecycle races under `-race` — timer flush vs concurrent `Execute` (workspace lock serializes, no deadlock); `Close()` vs armed timer (cancel + flush + join, no goroutine leak); failed pre-durable flush fails the structural command and retains pending for retry; two workspaces flush independently.

### Web: deck + frame rendering (`os-window` suites + new deck suites)

- **UT-070** (happy): frame with 2 members renders the deck with traffic lights inside it and none in the head; 1 member → no deck, controls in head.
- **UT-071** (state): adding then removing a second tab toggles deck mount/unmount with a single controls cluster at each state.
- **UT-072** (state): sole pinned survivor renders as a normal single window; regrouping shows it pinned again.
- **UT-073** (happy): switching tabs swaps head slot content (root identity vs breadcrumb vs document), strip presence, and keeps each body's DOM alive (inactive hidden, state preserved).
- **UT-074** (state): activating a suspense-pending tab shows its loading body immediately; deck stays interactive.
- **UT-075** (state): tab switch (and tab close) during an active drag session cancels the gesture (`gestureCancelled`) with no command dispatched.
- **UT-076** (state): keyboard activation while pointer hovers another tab activates the keyboard target.
- **UT-077** (boundary): 30 tabs compress to the 96px floor then scroll; every tab reachable by scroll; badge of an off-screen tab still counted by dock aggregate.
- **UT-078** (happy): 300-char emoji/RTL title truncates by width with full-path tooltip; two same-leaf tabs get distinct tooltips.
- **UT-079** (state): new-tab window renders "New tab" label and empty-state body with picker CTA.
- **UT-080** (happy): ⌘T on a focused frame dispatches `window.open{stack_target}` with app `new-tab` and activates it; repeated ⌘T creates multiple.
- **UT-081** (state): dismissing the picker keeps the new-tab window; ⌘W closes it.
- **UT-082** (state): ⌘T with no focused window opens a standalone new-tab window.
- **UT-083** (happy): dock app context menu offers "Open in new window" / "Open as tab in focused window" / "Open new instance"; each dispatches the right command.
- **UT-084** (happy): context menu on an open item shows "Go to tab" which focuses+activates; disabled "Open as tab" when no window focused states why.
- **UT-085** (state): context menu closes/never opens over a modal overlay.
- **UT-086** (happy): menu opens via keyboard (menu key/long-press equivalent) and all items reachable by arrows+Enter.
- **UT-087** (state): reopened/restored tab whose session query 404s renders the truthful gone-notice with a path to the parent list.
- **UT-088** (happy): "Close other tabs" dispatches ONE `window.close{scope:"others"}`; "Close tabs to the right" dispatches ONE `window.close{scope:"right"}`; ⌥-click × = the others scope; all-pinned → items disabled (no dispatch).
- **UT-089** (state): activating a tab whose drilled entity was deleted shows gone-state with back path.
- **UT-090** (happy): palette Go-to-tab lands at the tab's current depth (route + navstack untouched).
- **UT-091** (happy): label follows leaf live: root shows app name; after drill-in shows leaf; after pop reverts.
- **UT-092** (state): no rename affordance in deck/tab context menu/palette (assert absence).
- **UT-093** (happy): deck tab drag-out beyond threshold dispatches ungroup with pointer-derived floating rect; origin activates neighbor; new window focused.
- **UT-094** (state): drag interrupted (Escape/pointercancel/disconnect simulation) dispatches nothing; deck order restored.
- **UT-095** (happy): dragging a window over a deck shows the insertion indicator at pointer index (auto-scrolling at edges); drop commits ONE `window.stack.group{insert_index}` at exactly that index; drop elsewhere cancels with no dispatch.
- **UT-096** (happy): "Merge all windows" enabled at ≥2 windows on the desktop, dispatches one `window.stack.group` with all ids; "Move tab to new window" dispatches ungroup.
- **UT-097** (happy): tab state rendering — running dot pulse, needs-input accent badge with count, idle/done quiet; selection styling carries zero accent.
- **UT-098** (state): rapid state flaps render the final state (no queued animation storm).
- **UT-099** (state): shortcuts ignored when target is editable (input/textarea/contenteditable); `event.repeat` ignored.
- **UT-100** (happy): palette lists "Go to tab" results with window/desktop context and attention state; identical labels disambiguated; zero matches → group absent.
- **UT-101** (happy): selecting a minimized/other-desktop tab result un-minimizes, switches desktop, focuses, activates.
- **UT-102** (state): tiled pane deck renders inside pane bounds; narrow pane compresses then scrolls; per-pane decks operate independently.
- **UT-178** (boundary): group projection at volume — 200 windows across 12 groups projects correctly (frames, decks, active members, order) in a single pass with no per-member quadratic recomputation (assert projection call counts/shape, not wall-clock), keeping the PRD hundreds-of-tabs claim honest at the projection level.

### Go: listing + contract (`service_contract_test.go`, `contract_test.go`)

- **UT-110** (happy): wire snapshot exposes `floating_stacks`, `nav_stack`, `pinned`, `closed_entry_count`; zero-group workspace yields empty arrays/zero count (fields present).
- **UT-111** (happy): `ClientView` wire carries `stack_active`; only the owning client's frame includes it.
- **UT-112** (state): wire snapshot has NO `history` field (marshal round-trip asserts absence); domain History still drives undo.
- **UT-113** (happy): each new command decodes strictly (unknown field → 422 `window_manager_invalid_command`), executes, returns v3 snapshot; `navigate{mode:"pop"}` with a non-empty `route` and `close{minimize:true, scope:"group"}` both reject deterministically.
- **UT-114** (error): `set_active`/`reorder` on non-stacked → `window_manager_not_stacked`; on missing window → `window_manager_window_not_found`; close pinned → `window_manager_window_pinned`; stale revision → `window_manager_revision_conflict` (exact code strings + HTTP 422/404/409 mapping).
- **UT-115** (boundary): revision above `WindowManagerMaxSafeRevision` still rejected (regression with new fields).

### Go: hooks (`hooks/window_manager_dispatch_test.go`)

- **UT-120** (happy): `window.open`/`window.reopen` dispatch `window_manager.window.opened`; `window.close` dispatches `.closed` (payload lists all closed window ids for a frame close).
- **UT-121** (happy): `window.stack.group` dispatches `.stack.grouped` once with the stack's NodeID; an ungrouping move/toggle dispatches `.stack.ungrouped`; merge-all emits ONE grouped event for one revision.
- **UT-122** (state): `window.stack.set_active` and each coalescer flush dispatch exactly one `window_manager.stack.activated` (workspace/stack/window/actor/revision in payload); raw `window.focus` presentation flips dispatch nothing; introspection lists the 5 new events with payload schemas.

### Go: config (`config_test.go` + `internal/config/window_manager_test.go`)

- **UT-140** (happy): defaults `nav_stack_limit=50`, `closed_entry_limit=20`; TOML overrides parse and win.
- **UT-141** (error): `nav_stack_limit=0`/`201`, `closed_entry_limit=0`/`101` → `ValidationError` naming the path and range.
- **UT-142** (happy): workspace overrides merge via the pointer-override pattern like existing keys.
- **UT-143** (happy): `CanonicalShortcuts` accepts `window.tab.new|next|previous|last|reopen|jump.1..8`, rejects `window.tab.jump.9`, detects chord conflicts against existing actions.

### Go: CLI (`cli/window_manager_test.go`)

- **UT-150** (happy): `compozy window group --target w-a --windows w-b,w-c --insert-index 1 -o json` sends `window.stack.group{insert_index:1}` and prints the result bundle with stacks visible.
- **UT-151** (happy): `compozy window activate w-b` sends `window.stack.set_active`; `--client c1` also carries the client id.
- **UT-152** (happy): `compozy window pin w-a` / `unpin w-a` map to `window.pin{pinned}`; `compozy window reopen` maps to `window.reopen` and prints `applied=false` on empty history; extended flags round-trip: `window open --stack-target w-a`, `window navigate --mode push|pop` (route flags rejected with `pop`), `window close --scope others|right|group`.
- **UT-153** (happy): `compozy window list -o json` includes stack id, member order, active, pinned, nav depth per window.
- **UT-154** (error): CLI surfaces deterministic errors verbatim (`window_manager_not_stacked` etc.) with non-zero exit.

### Go: native tools (`daemon/native_tools_test.go`)

- **UT-160** (happy): `compozy__window_{group,reorder,activate,pin,reopen}` exist exactly once in toolset `compozy__window_manager` with descriptors, I/O schemas, and availability gates (WindowManager+Workspaces deps), and map payload→command exactly like the CLI; the updated `compozy__window_{open,navigate,close}` schemas expose stack-target/mode/scope. No global catalog-count assertion — catalog completeness is owned by the generated-catalog/codegen gates.

### packages/ui: context-menu + topbar

- **UT-170** (happy): context-menu is exported from `@compozy/ui` index, opens on contextmenu event, fires item `onSelect`, supports disabled items + separators + shortcuts labels (story present; no-shadow lint passes).
- **UT-171** (state): `TopbarSlotValue` has no `nav` field (type-level + runtime: publishing legacy nav content is impossible); toolbar renders the leading views group per the strip order.

### Web: D3 strip migration (route component suites)

- **UT-130** (happy): tasks catalog publishes List|Kanban as the toolbar's leading pill-group (views · filters · spacer · Rows|Cards) and nothing in the head; marketplace kind page likewise.
- **UT-131** (state): a route with no peer views renders no views group; strip absent when no toolbar content at all.

## Integration Tests

- **IT-001**: Manager + real clientstate store — group two windows, push nav, pin, close one, reopen; reload a fresh Manager from the same store; assert full v3 round-trip (membership/order/pins/navstacks/closed count/last-active).
- **IT-002**: Repository + store discard classes — a seeded v2 blob and a malformed/unknown-field blob each log, delete the row, and reinitialize; a v3 blob with a workspace-ID mismatch or a current-version topology validation failure hard-errors WITHOUT deleting the stored blob; a valid v3 blob loads untouched.
- **IT-003**: HTTP `POST /commands` `window.stack.group` end-to-end (httptest daemon): 200 with v3 snapshot; UDS byte-parity for the same request (transport-parity harness).
- **IT-004**: Two API clients race group/close on one revision — exactly one 200, one 409 `window_manager_revision_conflict`; rebase-guarded retry succeeds.
- **IT-005**: Close → reopen over HTTP preserves session tab identity: close a session-hosting window, assert session state endpoint unaffected, reopen restores route+navstack; reopen works from a second client (history is workspace-level).
- **IT-006**: WS stream — grouping/closing emits event frames with bumped revision to all subscribers; `window.focus` stack activation emits a client frame only to the acting client; needs-input attention flows via the sessions surface regardless of tab presence.
- **IT-007**: Two registered clients — A focuses tab X, B focuses tab Y in the same stack; ClientViews diverge; A's structural close of Y repairs B's view (client frame); a re-registered client seeds from durable last-active.
- **IT-008**: Native tool → command loop: `compozy__window_group` through the tool runtime mutates topology identically to the HTTP path (same result bundle).
- **IT-009**: `make codegen` contract drift — OpenAPI schema contains the new command payloads/fields, `history` absent from the snapshot schema, TS types compile in web (`codegen-check` clean).
- **IT-010**: CLI ↔ HTTP behavior parity — `compozy window group|activate|pin|reopen` against a live daemon produce the same snapshots/errors as raw HTTP; malformed `--windows` list surfaces the validation error.
- **IT-011**: Hook dispatch integration — a script hook registered for the 5 new events receives exactly one payload per user-level operation (merge-all → one grouped; a burst of tab switches → one activated per coalesced flush), and none for raw presentation focus flips.
- **IT-012**: Layout profile round-trip — export v3 with a floating stack + pins, PUT as profile, apply on a fresh workspace, assert reconstruction; v2 profile apply → 422 `window_manager_invalid_topology` with `topology.unsupported_version`.

## End-to-End Tests

Browser (`os-shell.spec.ts` extensions) unless noted:

- **E2E-001** (US-001, US-014): drag one floating window onto another's deck region → insertion affordance → drop → one frame, two tabs, dropped tab active; drag away cancels without merging.
- **E2E-002** (US-002, US-024): `layout.arrange stack` via CLI on two tiled windows → pane shows deck; close one tab → deck unmounts, controls return to head; both members were pointer-reachable.
- **E2E-003** (US-003, US-012): frame with Tasks-root, drilled Run, session doc — clicking tabs swaps root head / breadcrumb head / document head; composer draft survives a switch away and back.
- **E2E-004** (US-004): 12-tab frame compresses to floor then scrolls; hover tooltip shows full path.
- **E2E-005** (US-005, US-020): ⌘T mounts deck with New tab + picker; pick Tasks; ⌃Tab cycles; ⌘3 jumps; ⌘9 lands on last; ⌘[ pops the drilled tab only.
- **E2E-006** (US-006): right-click dock app → "Open as tab in focused window" adds the tab; keyboard-only path produces the same result.
- **E2E-007** (US-007, US-008): ⌘W closes a running session tab silently; dock badge persists; ⌘⇧T restores it into the frame at its depth; second ⌘⇧T restores the previously closed one (reverse order) after a full page reload.
- **E2E-008** (US-009, US-010): pin a tab → collates left, no ×, ⌘W refused; "Close other tabs" spares it; unpin then close works.
- **E2E-009** (US-011): drill Tasks→detail→run inside one tab; siblings unmoved; breadcrumb back pops one level; URL follows the focused tab's route.
- **E2E-010** (US-013): tear a tab out beyond the deck → standalone window at pointer with stack preserved; origin pane stays valid.
- **E2E-011** (US-018, US-019, US-026.EC-2): needs-input session tab badges; agent (CLI) closes it; operator's frame updates live; dock still routes to the session; answering clears everywhere.
- **E2E-012** (US-015): palette "Merge all windows" combines the desktop's three windows into one frame (order stable); "Move tab to new window" splits one back out.
- **E2E-013** (US-016, US-017): open second Tasks instance via context menu; different filters coexist; dock click cycles instances; 100×-lite volume (15 tabs) stays responsive.
- **E2E-014** (US-021, US-017.EC-1): ⌘K "Go to tab" from desktop 1 reaches a tab on desktop 2 (switch+focus+activate in one action); result rows disambiguate identical labels.
- **E2E-015** (US-022): build groups/pins/drilled tabs; restart daemon + reload browser; everything restores including last-active per frame.
- **E2E-016** (US-023): two browser contexts on one workspace hold different active tabs in the same frame; membership edits replicate live.
- **E2E-017** (US-025, US-026, US-028 — CLI journey, runtime harness): `compozy window list -o json` shows tab topology; `group→activate→pin→reopen` sequence via CLI drives the daemon with deterministic errors on a stale revision; `compozy layout export|apply` round-trips the grouped layout.
- **E2E-018** (US-029): Tasks catalog and Marketplace render views in the strip (screenshot-verified), head two-element in root and drill-in states.
