# Test Specification: AGH OS Shell (`os-shell`)

Canonical test contract for the OS shell program. Companion to `_techspec.md`.
Derived from `_user_stories.md` (behavior) and `_techspec.md` (components — §Routing Model, §Attention Surfaces, §Presentation Modes, §Safety Invariants). ID history: first draft UT-001..062/IT-001..007/E2E-001..014; story sync added UT-063..069, E2E-015..016; peer-review round 1 (B-010) added UT-070..084, IT-008..009, E2E-017..023 and corrected mis-mapped citations; the modal-policy addendum (US-020) added UT-085..086, E2E-024; Task 01 contract repair added UT-087..089; the task-08 window-head batch minted UT-090..094; the ADR-009 snap amendment added UT-095..UT-101 and E2E-025..E2E-027 (invariant 19, US-021). Invariant numbers reference the post-round-1 Safety Invariants list.

## Strategy

- **Go**: table-driven `t.Run("Should …")` + `t.Parallel()` per `eng-test-conventions`; real bbolt in `t.TempDir()` (fakes at I/O boundaries only — there is no I/O boundary to fake below the store); WS handlers exercised over `httptest.Server` with a real `gorilla/websocket` client; `-race` mandatory. Integration cases build the daemon-wired stack under the `+integration` tag.
- **Web**: Vitest + Testing Library (jsdom) via `bunx turbo run test --filter=./web`; `OsStateClient` takes an injectable socket factory (mirrors the existing `eventSourceFactory` pattern); store tests are pure TS. Storybook stories + `eng-ui-screenshot` own pixel evidence (not asserted here — visual QA is capture-gated, not test-gated).
- **E2E**: Playwright (`make test-e2e-web`) against the daemon-served SPA; journeys go through the public UI exactly as an operator would; CLI-parity journey shells out to `agh desktop-state`.
- **Conventions**: every API case asserts status **and** body (code + payload); unit error cases assert sentinel/`errors.Is` identity; IDs below are permanent once tasks reference them.

## Coverage Matrix

Story rows (from `_user_stories.md`) first, then component rows (from `_techspec.md`).

| Source | Behavior | Unit | Integration | E2E |
|---|---|---|---|---|
| US-001 (+EC-2/3/4) | Open from dock; idempotent; hint; unknown entries dropped; soft cap | UT-040, UT-041, UT-063, UT-064 | — | E2E-001, E2E-002 |
| US-002 (+EC-1/2) | Drag/resize within bounds; viewport re-clamp | UT-065 | — | E2E-002, E2E-003 |
| US-003 (+EC-1/2) | Zoom/minimize/restore/close; keyboard; stream survives minimize | UT-045–UT-047 | — | E2E-003, E2E-004, E2E-006 (minimize-while-streaming step) |
| US-004 (+EC-1/3) | N session windows side by side, independent | UT-042 | — | E2E-006, E2E-012 |
| US-005 (+EC-1/2) | In-window nav, breadcrumb, back semantics | UT-057, UT-058, UT-080 | — | E2E-005, E2E-007 |
| US-006 (+EC-1/2) | Badges bound to named projections; zero hidden; cap; stale hides | UT-066, UT-082 | — | E2E-020 |
| US-007 (+EC-1/2/4, AC-5) | Rail recent/all, filter, groups, toggle, compact overlay | UT-067, UT-068, UT-084 | — | E2E-006, E2E-020 |
| US-008 (+EC-1/2) | Bell aggregator: waiting items, focus action, live removal, disconnect | UT-069, UT-083 | — | E2E-015 |
| US-009 (+EC-1/2/3) | ⌘K palette everywhere; ⌘J runtime picker; overlay unwinding | UT-059, UT-060 | — | E2E-008, E2E-017 |
| US-010 (+EC-1/2) | Spaces per workspace; overview; purge on delete | UT-076, UT-077 | IT-004 | E2E-009 |
| US-011 (+EC-1/2/3) | Exact idempotent restore; daemon restart survival | UT-044, UT-063 | IT-005 | E2E-002, E2E-003, E2E-005 |
| US-012 (+EC-1/2) | Live convergence, same daemon; LWW by seq; simultaneous same-window edit | UT-019, UT-052–UT-055, UT-070, UT-074 | IT-001, IT-002, IT-009 | E2E-010, E2E-018 |
| US-013 (+EC-1/2) | Degraded fully functional; recovery pushes touched keys | UT-056, UT-075 | — | E2E-012, E2E-019 |
| US-014 (+EC-1/2, AC-3) | Compact stack, tab-bar dock; geometry preserved; deep link + attention parity | UT-061 | — | E2E-011, E2E-020 |
| US-015 (+EC-1) | Wallpaper per space; system reduced-motion precedence | — | — | E2E-013, E2E-021 |
| US-016 (+EC-1/2/3) | URL meaning, deep links, focus history, cross-workspace switch | UT-057, UT-080, UT-081 | — | E2E-005, E2E-007, E2E-016 |
| US-017 (+EC-1/2/3/4) | Agent inspects/arranges desktop; deterministic errors incl. workspace_not_found | UT-006, UT-007, UT-032, UT-035–UT-039, UT-076 | IT-006 | E2E-014 |
| US-018 (+EC-1/2) | Documented deterministic surface; CLI/API/UDS parity incl. watch | UT-020, UT-029 | IT-007, IT-008 | — |
| US-019 (+EC-1/2/3) | Menubar menus, workspace trigger, popover discipline | UT-059 | — | E2E-022, E2E-017 |
| US-020 (+EC-1/2/3) | Window-scoped dialogs; minimize protection; stacking | UT-085, UT-086 | — | E2E-024 |
| US-021 (+EC-1/2/3/4) | Snap to halves/quarters; overlay; derived reflow; restore; agent snap | UT-095–UT-101 | — | E2E-025–E2E-027 |
| Perf envelope (Assumptions) | 12-window fluidity, restore latency, convergence | — | — | E2E-023 |
| clientstate store/service | CRUD, rev/seq, CAS, tombstones, limits, isolation, durability | UT-001–UT-021, UT-070–UT-073, UT-076–UT-077, UT-087–UT-089 | IT-003, IT-005, IT-009 | — |
| clientstate hub/watch | fan-out, commit-order, linearized subscribe, eviction, origin | UT-013–UT-016, UT-070, UT-071 | IT-001, IT-002, IT-009 | — |
| Config `[desktop_state]` | defaults, global overrides, workspace rejection, invalid values | UT-020 | — | — |
| API HTTP handlers | success + every failure shape incl. workspace_not_found + apply | UT-022–UT-029, UT-076 | IT-001, IT-002, IT-007 | — |
| API WS endpoint + pumps | sub/snapshot/apply/ack/event/error/close; deadlines; shutdown | UT-030–UT-034, UT-078, UT-079 | IT-001, IT-002, IT-008 | E2E-010, E2E-012 |
| CLI `agh desktop-state` | verb family, `-o` output, errors, UDS watch | UT-035–UT-039 | IT-006, IT-008 | E2E-014 |
| Daemon wiring | deletion gate + purge, boot/shutdown joins | UT-079 | IT-004, IT-005 | — |
| Workspace isolation | no cross-leak, no resurrection | UT-021, UT-076, UT-077 | IT-003 | — |
| Desktop store (WM) | invariants 10, 12, 14; lifecycle; batches | UT-040–UT-049, UT-063–UT-065, UT-073 | — | E2E-002–E2E-004 |
| App registry | instance match, path ownership | UT-050, UT-051 | — | — |
| OsStateClient | invariants 3, 6, 15, 16; pending mutations | UT-052–UT-056, UT-074, UT-075 | — | E2E-010, E2E-012, E2E-019 |
| Routing coordinator (URL↔WM) | causes, single-write history, hydrate precedence, keyboard | UT-057, UT-058, UT-080, UT-081 | — | E2E-005, E2E-007, E2E-016 |
| Palette + shortcuts | ⌘K takeover, ⌘J rebind, sources | UT-059, UT-060 | — | E2E-008, E2E-017 |
| Presentation modes | floating/compact derivation + gating | UT-061 | — | E2E-011, E2E-020 |
| Dock, rail, bell | projections, rail model, aggregator | UT-062, UT-066–UT-069, UT-082–UT-084 | — | E2E-015, E2E-020 |
| Window snap layer | invariant 19: zones, exclusivity, derived geometry, no-commit-on-resize, codec salvage | UT-095–UT-101 | — | E2E-025–E2E-027 |

Coverage notes: US-004.EC-2 (terminal session state rendered truthfully inside its window) is owned by the existing session-view suites — the shell rehosts those views unchanged; no shell-layer duplicate is added. The nav-counts store's existing 8-key suite remains its own canonical owner; UT-082 covers only the dock badge adapter's two named projections.

## Unit Tests

### `internal/clientstate` — store + service (TechSpec: Core Interfaces, Data Models)

- **UT-001** (happy): `Put` new key `os_shell/desktop` in ws `w1` — returns `Entry{Rev:1}`; `Get` returns identical value/rev/updatedAt.
- **UT-002** (state): second `Put` on the same key — `Rev` becomes exactly 2; payload replaced.
- **UT-003** (error): `Get` on an absent key — `errors.Is(err, ErrNotFound)`.
- **UT-004** (happy): `Put` with `IfRev` equal to current rev — succeeds, rev increments.
- **UT-005** (error): `Put` with stale `IfRev` — `ErrRevConflict`; stored value and rev unchanged.
- **UT-006** (boundary): payload of exactly `max_value_kib` KiB passes; one byte over — `ErrValueTooLarge`.
- **UT-007** (boundary): with `max_keys_per_workspace=512`, the 513th retained key identity across all domains — including tombstones — fails `ErrKeyQuota`; an update or tombstone recreation of an existing identity still succeeds.
- **UT-008** (error): domain `"OS Shell"` (uppercase/space) — `ErrInvalidDomain`.
- **UT-009** (error): key with `/` or 129 chars — `ErrInvalidKey`.
- **UT-010** (happy): `List` returns all entries of a domain with values and revs; other domains excluded.
- **UT-011** (state): `Delete` existing key — subsequent `Get` is `ErrNotFound`; watcher receives `Event{Deleted:true}`.
- **UT-012** (error): `Delete` with stale `IfRev` — `ErrRevConflict`, entry intact.
- **UT-013** (ordering): watcher receives put/put/delete for one key in strictly increasing rev order.
- **UT-014** (happy): three subscribers on one workspace all receive the same event.
- **UT-015** (concurrency): subscriber that never drains — after buffer (256) overflows it is evicted, `Err()` is `ErrSlowConsumer`, and the other subscriber still receives every event.
- **UT-016** (happy): `Put` with `Origin:"conn-a"` — delivered `Event.Origin == "conn-a"`.
- **UT-017** (state): `PurgeWorkspace` — all domains/keys gone, workspace's subscriptions closed, other workspace untouched.
- **UT-018** (state): close store, reopen same file — entries and revs preserved (durability).
- **UT-019** (concurrency): 8 goroutines × 50 `Put`s across distinct keys under `-race` — every key ends at rev 50, no lost updates.
- **UT-020** (boundary/error): config defaults load (`256`, `512`); `max_value_kib=0` and `=5000`, plus `max_keys_per_workspace=15` and `=8193`, fail validation with named-key errors.
- **UT-021** (state): watcher on ws `w1` receives nothing from `Put`s on ws `w2`.

### `internal/clientstate` — sequencer, apply, tombstones, resolver (round 1: invariants 1–4, 9)

- **UT-070** (concurrency): 4 goroutines applying to distinct keys under `-race` — two independent subscribers receive **identical seq-ordered streams**, and delivery order equals commit order for every prefix (invariant 2).
- **UT-071** (concurrency): subscribe racing a write burst — the subscription's `AsOfSeq` fences exactly: no delivered event has `seq ≤ AsOfSeq`, no committed event with `seq > AsOfSeq` is missing, including a key created between snapshot and first event (invariant 3).
- **UT-072** (state): delete then recreate a key — the tombstone preserves rev continuity (recreate rev = tombstone rev + 1); CAS with the pre-delete rev fails `ErrRevConflict`; tombstones never appear in `List`/`Snapshot` (invariant 1).
- **UT-073** (state): 3-op `Apply` where op 3 has a stale `IfRev` — nothing commits (all-or-nothing); a valid 3-op `Apply` commits with consecutive seqs and returns all three entries (invariant 4).
- **UT-074** (state, web): pending-mutation machine — a remote event on a key with an in-flight `apply` (`req` pending) is buffered; after the ack, the higher-seq value settles with no oscillation (invariant 6).
- **UT-075** (state, web): degraded recovery — keys modified while degraded are re-applied as one batch after snapshot adoption; untouched keys take the snapshot value (invariant 10; US-013.AC-2).
- **UT-076** (error): every service operation (`Get`/`List`/`Apply`/`Watch`) against an unknown or deleting workspace id, and `Purge` without a matching deletion-gate generation — `ErrWorkspaceNotFound`; nothing is created implicitly (invariant 9; US-017.EC-4).
- **UT-077** (concurrency): deletion gate — `Apply` racing `PurgeWorkspace` either commits before the gate closes or fails `ErrWorkspaceNotFound`; subscriptions close; purge is idempotent while the generation remains delete-gated; a delayed old-generation apply cannot write after same-id recreation, which starts empty (invariant 9; US-010.EC-1).
- **UT-087** (error): `OpPut` with malformed JSON or a valid JSON scalar/array — `ErrInvalidValue`; the batch changes nothing (invariant 11).
- **UT-088** (error): an empty `Apply` — `ErrEmptyApply`; workspace sequence and stored state remain unchanged.
- **UT-089** (lifecycle): after service close, `Get`/`List`/`Apply`/`Watch`/`PurgeWorkspace` return `ErrClosed`; closing a subscription or the service again is safe and idempotent.

### `internal/api` — desktop-state handlers (TechSpec: API Endpoints)

- **UT-022** (happy): `GET .../desktop-state/desktop` — `200` with `{key,value,rev,seq,updated_at}`.
- **UT-023** (error): `GET` absent key — `404`, body `code=desktop_state_not_found`.
- **UT-024** (happy): `PUT` valid body — `200`, returned `rev` incremented.
- **UT-025** (error): `PUT` with stale `if_rev` — `409`, `code=desktop_state_rev_conflict`.
- **UT-026** (error): oversized `PUT` — `413`, `code=desktop_state_value_too_large`.
- **UT-027** (error): quota-exceeded / bad key `PUT` — `422` with `desktop_state_key_quota_exceeded` | `desktop_state_invalid_key` respectively (no public domain axis — ADR-008).
- **UT-028** (state): `DELETE` existing — `204`; absent — `404`; stale `if_rev` — `409`.
- **UT-029** (happy): `GET .../desktop-state` list — `200` `{as_of_seq,entries:[...]}` sorted by key.
- **UT-030** (happy): WS `sub` — exactly one `snapshot` frame carrying current entries and its `as_of_seq` fence.
- **UT-031** (happy): WS one-op `apply` — sender gets `ack{req,results:[{key,rev,seq}]}`; a second connection gets `event` with `origin` = sender's conn id.
- **UT-032** (error): WS one-op `apply` oversized — `error{req,code:desktop_state_value_too_large}`; connection stays open and usable.
- **UT-033** (concurrency): non-draining WS client — server sends `error{code:desktop_state_slow_consumer}` then closes that socket only.
- **UT-034** (state): WS connection closed immediately after an `apply` frame is read — the write still commits (daemon-owned lifecycle context, invariant 8); verified via HTTP `GET`.
- **UT-078** (concurrency): writer-pump discipline over real sockets — concurrent acks (from the conn's own applies) and events (from another writer) serialize through the single pump; a stalled reader hits the write deadline and is evicted without delaying a healthy subscriber's next event (invariants 5, 8).
- **UT-079** (state): daemon shutdown with a mutation queued behind the sequencer — the mutation completes or cancels cleanly (never half-commits), both pumps join, and the store closes last; no goroutine leak under `-race` (invariant 8).

### `internal/cli` — `agh desktop-state` (TechSpec: API Endpoints / Agent Manageability)

- **UT-035** (happy): `set --value '{"a":1}' -o json` then `get -o json` — identical value, rev present, valid JSON output shape.
- **UT-036** (error): `set --if-rev 1` when rev is 3 — non-zero exit, structured error `code=desktop_state_rev_conflict`.
- **UT-037** (happy): `list -o json` — array of entries for the domain.
- **UT-038** (happy): `watch -o jsonl` — emits one JSON line per event for a concurrent `set`/`delete`, in seq order, parseable.
- **UT-039** (error): `get` absent key — non-zero exit, `code=desktop_state_not_found`.

### `web/src/systems/os` — desktop store (TechSpec: Core Interfaces; invariants 10, 12, 14)

- **UT-040** (happy): `openOrFocus({app:"tasks"})` — window `app:tasks` created with registry default rect/location, focused, `z` = max.
- **UT-041** (state): second `openOrFocus({app:"tasks"})` — no new window; existing one focused (invariant 14).
- **UT-042** (happy): `openOrFocus({app:"session", instanceKey:"s1"})` then `"s2"` — two windows `session:s1`, `session:s2`.
- **UT-043** (state): `focusWindow` on a background window — its `z` becomes max; previous focus keeps lower `z`.
- **UT-044** (boundary): hydrating windows with `z` values `[7, 903, 12]` — compacted to `1..3` preserving order (invariant 12).
- **UT-045** (state): `minimizeWindow` — entry survives with `minimized:true` and rect intact; `restoreWindow` clears it and focuses.
- **UT-046** (state): `toggleZoom` — stores `prevRect`, sets `maximized`; second toggle restores exact `prevRect`.
- **UT-047** (state): closing the focused window — next-highest-`z` window gains focus; closing the last window — `focusedId` null.
- **UT-048** (state): `applyRemote` with `seq` ≤ the locally settled seq for that key — ignored; higher seq applies (invariant 10, LWW by seq).
- **UT-049** (state): `applyRemote` delete event for `win:app:tasks` — window removed locally.
- **UT-063** (error): hydration input containing an unknown app id, an unreadable payload, and two valid windows — invalid entries dropped individually; both valid windows restore (US-001.EC-2, US-011.EC-1/EC-2).
- **UT-064** (boundary): opening the 13th window — window opens AND the store emits the one-time soft-cap guidance signal; the 14th does not re-emit (US-001.EC-3).
- **UT-065** (boundary): viewport shrinks so a window's rect falls outside the desktop bounds — the window re-clamps to a reachable position; growing the viewport back does not move it again (US-002.EC-1/EC-2).

### `web/src/systems/os` — app registry

- **UT-050** (happy): `matchInstance("/agents/webgen/sessions/s1")` on the session app — returns `"s1"`; `/agents/webgen/settings` — null.
- **UT-051** (happy): pathname→app resolution — `/loop-runs/r1` maps to `loops`; `/marketplace/skills` to `marketplace`; unknown path maps to no app.

### `web/src/systems/os` — OsStateClient (invariants 3, 6, 15, 16; fake socket)

- **UT-052** (ordering): event with `seq` ≤ the snapshot's `as_of_seq` is dropped; higher seq applies (snapshot fence, invariant 3).
- **UT-053** (state): event whose `origin` equals this client's connection id is ignored (invariant 6).
- **UT-054** (boundary): three `commitRect` calls within 250ms — exactly one `apply` frame (trailing debounce); gesture-end flush sends immediately (invariant 15).
- **UT-055** (state): socket drop + reconnect — client re-`sub`s, applies fresh snapshot replacing stale local mirror per invariant 10.
- **UT-056** (error): socket factory rejects — store `hydration:"degraded"`; WM actions still mutate local state; retry scheduled with backoff (invariant 16).

### Routing coordinator + controllers (TechSpec: Routing Model; invariant 13)

- **UT-057** (happy): tasks sync-controller mounted at `/tasks/t9` — reports the location to the coordinator; store shows the tasks window at that location; renders null.
- **UT-058** (error): tasks controller given `search {mode:"bogus"}` — shared zod schema falls back to default mode without throwing.
- **UT-080** (state): coordinator causes — a `route-pop` never calls `navigate` (no history write); a `user-focus` writes exactly one entry; `hydrate` applies the snapshot first and then the initial URL wins focus while the restored desktop survives (Routing Model rules 2–4).
- **UT-081** (happy): keyboard activation (Tab/Enter into a focusable inside an unfocused window) triggers `user-focus` with the same semantics as pointerdown (Routing Model rule 5).

### Palette + shortcuts (ADR-005)

- **UT-059** (happy): palette lists app entries, session entries (label + agent meta), and actions; typing filters; Enter fires `navigate` for the selection.
- **UT-060** (state): ⌘K keydown anywhere opens the palette; ⌘J fires RuntimeSelector open only when focus is inside the visible, enabled selector's nearest form or dialog.

### Presentation + dock + rail + bell

- **UT-061** (state): viewport 959px — store `presentation:"compact"`; `commitRect`/`toggleZoom` become no-ops; back to 1200px — rects intact.
- **UT-062** (happy): dock badge adapter — nav-counts `{sessions:1, tasks:2}` renders badges on exactly those dock items.
- **UT-066** (boundary): badge count 0 renders nothing (no "0"); count 12 renders the capped display ("9+") while remaining distinguishable from zero (US-006.EC-1/AC-2).
- **UT-067** (happy): rail filter with query `"web"` narrows rows to sessions whose title or agent matches, live per keystroke; clearing restores the full list (US-007.AC-3).
- **UT-068** (state): collapsing an agent group in the rail's full view persists; re-rendering the rail keeps the group collapsed (US-007.AC-2).
- **UT-069** (state): bell aggregator model — a waiting session + an awaiting-approval task show count 2 and both rows with context; an item resolved at its source leaves the list live; zero shows "nothing waiting" (US-008.AC-1/AC-3/EC-1).
- **UT-082** (happy): dock badge adapter — waiting-sessions count derives from a session-catalog fixture; tasks badge equals the dashboard fixture's `awaiting_approval_tasks`; a stale source hides that badge entirely (US-006; invariant 17).
- **UT-083** (state): bell selection focuses the owning window (session window for a waiting session; tasks window for an awaiting task); with the daemon connection down, the popover shows the disconnect state instead of rows (US-008.AC-2/EC-2).
- **UT-084** (state): `toggleRail` flips and persists `railOpen`; in compact presentation the rail state renders as the overlay-sheet variant and dismissal restores the prior focus (US-007.AC-5/EC-4).

### Modal & overlay policy (TechSpec: Modal & Overlay Policy; invariant 18)

- **UT-085** (state): a `Dialog` rendered inside an `OsWindow` mounts within that window's `OverlayContainerContext` container (not `document.body`); the same component outside a window falls back to `document.body` (US-020.AC-1).
- **UT-086** (state): `minimizeWindow` on a window with an open modal dialog — the window hides but stays mounted (dock indicator set); closing the dialog completes the unmount; restore before close shows the dialog with its form state intact (US-020.EC-1; invariant 18).

### Unified window head (pagehead-redesign.html; US-005)

IDs continue after clientstate UT-087–089 (do not collide).

- **UT-090** (happy): a catalog window under `OsWindowFrame` with slot title published — the head renders the title once; the body has no `[data-slot="page-head"]`.
- **UT-091** (state): publishing `toolbar` on the topbar slot renders a context strip under the head; clearing `toolbar` removes the strip (two bars maximum).
- **UT-092** (happy): drill-in slot with `crumbs` + `onBack` — head shows back + parent crumb buttons + leaf title; crumb labels never include a workspace prefix; invoking back fires `onBack` once.
- **UT-093** (happy): document/session head publishes a state leading mark + self-title with empty crumbs — no breadcrumb trail and no route glyph tile.
- **UT-094** (state): scrolling the window body past 2px sets the head scrolled/elevated state; scrolling back to top clears it.

### `web/src/systems/os` — window snap layer (ADR-009; invariant 19; US-021)

- **UT-095** (happy): `snapWindow(id, {fx:0,fy:0,fw:.5,fh:1})` — `snap` set, pre-snap rect stored in `prevRect`, `maximized` cleared; the derived-geometry selector returns work-area × fractions for the local viewport.
- **UT-096** (state): exclusivity — `toggleZoom` on a snapped window clears `snap` and zooms; `snapWindow` on a maximized window clears `maximized`; each restore path returns the stored `prevRect` exactly.
- **UT-097** (state): any `commitRect` on a snapped window (drag-away/resize) clears `snap` in the same mutation and commits floating geometry; the drag-away path restores `prevRect` dimensions anchored under the pointer.
- **UT-098** (boundary): viewport resize with one snapped and one floating window — the snapped window's derived rect tracks the work area with **zero** persistence writes (spy on the client binder); the floating window only re-clamps per UT-065; a quarter zone deriving below 280×180 clamps to the minimum.
- **UT-099** (error): codec — a `win:*` doc with valid `snap` round-trips; out-of-range fractions (`fx:1.2`), negative sizes, or sub-minimum zones salvage `snap` to `null` while keeping the window (US-001.EC-2 posture); `snap` absent decodes as `null`.
- **UT-100** (happy): pure zone resolver — pointer positions within the 20px sensitivity radius of the left/right edges and each corner resolve their zone; center positions resolve none; exactly one hint is published per resolution (no dual targets at seams).
- **UT-101** (state): keyboard + palette — snap commands are listed for the focused window and dispatch `snapWindow`; ⌃⌥ chords dispatch the matching zone; in compact presentation every snap action is a no-op and the commands are absent (UT-061 gating).

## Integration Tests

### Daemon-wired desktop-state (`+integration`)

- **IT-001**: HTTP `PUT` `os_shell/win:app:tasks` → connected WS subscriber on the same workspace receives the matching `event` with correct rev and empty origin.
- **IT-002**: WS `put` from client A → HTTP `GET` returns the new value/rev; client B received the `event`, client A only the `ack`.
- **IT-003**: subscribers on workspaces `w1` and `w2` — writes to `w1` never produce frames on `w2`'s socket (isolation).
- **IT-004**: workspace deletion flow → `PurgeWorkspace` fires: `GET` returns 404, `w1` subscriber's socket closes; `w2` unaffected.
- **IT-005**: daemon restart with a populated store — entries and revs identical after reopen (durability through real boot path).
- **IT-006**: `agh desktop-state set` (CLI) → WS subscriber receives the event; CLI `get -o json` matches HTTP `GET` body (agent-operability parity, SD-011).
- **IT-007**: UDS transport — the CRUD + apply routes return byte-identical bodies to their HTTP twins for the same state.
- **IT-008**: UDS **watch parity** — the stream upgrade over the UDS listener delivers the same `as_of_seq` fence, the same seq-ordered events, and the same eviction/error codes as the HTTP-WS twin for an identical write sequence (peer-review B-008; US-018.AC-3).
- **IT-009**: commit-order under load — two writers (one HTTP, one WS) issue 100 interleaved applies; two subscribers (one WS, one UDS watch) observe one identical total order whose final state matches the store (invariants 2–3 end-to-end).

## End-to-End Tests

### Journey: daily desktop (open → arrange → persist)

- **E2E-001**: fresh boot → menubar, dock, and desktop render; empty desktop shows the ⌘K hint; no window is open.
- **E2E-002**: dock click on Tasks → window opens, URL becomes `/tasks`; drag the window by its head; reload the page → window reopens at the dragged position (daemon-persisted).
- **E2E-003**: resize via a corner handle → reload persists size; green traffic light zooms to fill; second click restores the exact prior rect.
- **E2E-004**: yellow traffic light minimizes → dock shows the minimized indicator; dock click restores the window with its content instantly.

### Journey: URL, deep links, focus history (ADR-002)

- **E2E-005**: navigate directly to `/tasks/<seeded-id>` → tasks window opens focused showing the detail; the window head shows a window-local breadcrumb trail (no workspace prefix) with the deep leaf as title; browser Back → window shows the list at `/tasks`.
- **E2E-007**: with tasks and agents windows open, click agents then tasks → URL follows each focus; browser Back refocuses agents without closing anything.

### Journey: multi-agent observation

- **E2E-006**: open two seeded sessions from the rail → two session windows side by side; both transcripts receive streamed events; replying in window 2 does not disturb window 1's scroll or focus; **minimize window 1 while it streams, restore it → the transcript shows everything that arrived while minimized** (US-003.EC-1).

### Journey: palette + keyboard

- **E2E-008**: ⌘K → type an app name → Enter opens its window; ⌘K → type a session title → Enter focuses that session window; ⌘K still opens the palette from that session's composer; in the New session dialog, focusing the RuntimeSelector host and pressing ⌘J opens the runtime picker.

### Journey: spaces + workspaces

- **E2E-009**: arrange windows in workspace A; switch to workspace B via menubar → different (empty) space; open one window; ⇧⌘S overview shows both spaces with mini-window thumbnails; clicking A restores A's exact arrangement.

### Journey: convergence + degradation (ADR-004)

- **E2E-010**: two browser contexts on the same workspace — moving a window in context A updates its position live in context B.
- **E2E-012**: block the WS route → shell loads in degraded mode with a non-blocking indicator; windows open/move normally; unblock → state resyncs and persists again.
- **E2E-014**: `agh desktop-state set` a new rect for `win:app:tasks` from the CLI → the open web desktop moves the window live (agent-arranged desktop, SD-011).

### Journey: attention (US-008)

- **E2E-015**: with a seeded pending approval — the menubar bell shows its count; opening the popover lists the approval; approving it applies the decision (observable on the target surface), removes the row, and clears the badge; resolving the same approval from the CLI first makes the UI action surface a truthful already-resolved error.

### Journey: cross-workspace deep link (US-016.EC-2)

- **E2E-016**: with workspaces A (active) and B — following a link to a session owned by B switches the shell to B's space and opens that session's window focused there; A's arrangement is intact when switching back.

### Journey: responsive + appearance

- **E2E-011**: viewport 390×844 → compact stack presentation (focused window fills viewport, dock bar, no drag handles); restore 1440×900 → floating windows return with prior rects.
- **E2E-013**: Appearance pane — switch wallpaper (persists across reload); enable reduce-motion → minimize happens without the genie animation.
- **E2E-021**: with the browser emulating `prefers-reduced-motion: reduce` and the in-product motion toggle set to full — animations stay reduced (system preference wins; US-015.EC-1).

### Journey: overlays + menubar (US-009.EC-2, US-019)

- **E2E-017**: open the bell popover, then press ⌘K — the palette opens and the popover closes; Escape closes the palette only; a second Escape returns focus to the shell (one layer at a time, no orphan popovers).
- **E2E-022**: menubar journey — the workspace trigger lists workspaces and switches; Session→"New session" opens a session flow; View→"Spaces overview" opens ⇧⌘S's overlay; Help lists shortcuts; the logo focuses the dashboard window; the cog opens settings.

### Journey: convergence + degradation, round-1 additions (US-012, US-013)

- **E2E-018**: two browser contexts drag the **same** window simultaneously — both settle on the identical final position with no oscillation between the two values (LWW by seq; US-012.AC-2).
- **E2E-019**: block the stream route → arrange three windows while degraded → unblock — the arrangement persists (reload proves it) and windows untouched while degraded adopt daemon truth (US-013.AC-2).

### Journey: compact parity (US-014.AC-3, US-007.EC-4)

- **E2E-020**: at 390×844, follow a deep link to a task detail — the tasks window opens focused in the stack; dock badges render truthfully in the tab bar; opening the sessions rail shows the overlay sheet and dismissing returns to the same window.

### Journey: window-scoped dialogs (US-020)

- **E2E-024**: with tasks and a streaming session window open — trigger a destructive confirm in tasks: the dialog dims only the tasks window; typing a reply in the session composer works while it is open; dragging the tasks window carries the dialog; confirming completes the action and releases only the tasks window.

### Journey: performance envelope (Assumptions / N-004)

- **E2E-023**: open 12 windows; drag one continuously for 3s — no frame longer than 50ms attributable to the shell (traced); reload — desktop restore (snapshot → all windows placed) completes within 500ms of the stream connecting; a third context's window move appears in the first two without layout thrash.

### Journey: window snap (US-021, ADR-009)

- **E2E-025**: drag the tasks window toward the right desktop edge — the zone overlay appears during the drag (and not before); release — the window fills the right half; reload — it restores snapped; resize the browser viewport — the window keeps filling the right half (derived reflow, no re-clamp jump).
- **E2E-026**: two browser contexts with different viewport sizes — snapping a window to the left half in context A renders it as the left half of context B's own viewport (fractions converge, px intentionally differ, no oscillation); then `agh desktop-state set` writes the `win:*` doc with `snap:{fx:.5,fy:0,fw:.5,fh:1}` — both contexts move it to the right half live (agent-arranged snap, SD-011).
- **E2E-027**: ⌘K → "Snap left half" snaps the focused window without pointer input; drag it away from the zone — its pre-snap size restores under the cursor; with `prefers-reduced-motion: reduce` emulated, the zone overlay renders with no fade/morph while snapping still works.
