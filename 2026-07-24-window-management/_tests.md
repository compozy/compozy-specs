# Test Catalog: Hybrid Window Management and Virtual Desktops

## Test doctrine

Tests assert behavior and public contracts, not implementation shape. Each invariant has one owning
layer and canonical suite. Generated artifacts are checked by codegen/build gates rather than frozen in
unit tests. Visual styling is checked through rendered Storybook capture and browser behavior, not CSS
literal assertions or snapshots.

## WM-GO — Domain service (`internal/windowmanager` canonical suite)

### WM-GO-001 — Initial snapshot

- **Invariant:** a new workspace has exactly one ordered standard desktop, revision zero/initial, no
  windows, and valid default overrides.
- **Owner:** window-manager service.
- **Evidence:** fresh load and reopen produce the same normalized state.

### WM-GO-002 — Normalization is deterministic and idempotent

- **Invariant:** empty nodes disappear, singleton containers collapse, same-axis splits flatten,
  weights normalize, stale/duplicate leaves resolve deterministically, and a second normalization is
  byte-equivalent.
- **Owner:** normalizer.
- **Evidence:** table cases plus deterministic randomized valid command sequences.

### WM-GO-003 — Topology validation

- **Invariant:** cycles, duplicate/cross-desktop leaves, unknown windows, overlapping/out-of-bounds
  group frames, invalid weights, invalid active stack IDs, and an absent final desktop are rejected.
- **Owner:** validator.
- **Evidence:** each error maps to its stable diagnostic code and makes no repository write.

### WM-GO-004 — One command, one transaction

- **Invariant:** every changing semantic command advances revision once and records one snapshot,
  event, and history entry; no-op/preview/rejection record none.
- **Owner:** service transaction coordinator.
- **Evidence:** repository spy and real repository integration cases.

### WM-GO-005 — Shared boundary resize

- **Invariant:** resizing `split_id + boundary_index` changes adjacent weights and therefore every
  descendant on both sides, including one-to-many arrangements, without breaking other adjacency.
- **Owner:** layout reducer.
- **Evidence:** left half beside two right quarters and nested variants.

### WM-GO-006 — Window lifecycle reflow

- **Invariant:** close/minimize removes a grouped leaf and normalizes; restore uses a valid return
  anchor or deterministic focus/float fallback; no other window leaves the desktop.
- **Owner:** window reducer.

### WM-GO-007 — Structural drag/drop and arrange

- **Invariant:** before/after/side/center drops and named arrangements include all explicit participants,
  never overlap group frames, and never leave an unmentioned partial mutation.
- **Owner:** layout reducer.

### WM-GO-008 — Floating/tiled transitions

- **Invariant:** toggle floating and drag-away preserve a clamped normalized floating rect and reflow
  the old group; modifier group move preserves the group topology.
- **Owner:** window/layout reducer.

### WM-GO-009 — Focus and MRU

- **Invariant:** click and directional focus use client-local active desktop and spatial/tree neighbors;
  closing focused selects deterministic MRU; non-wrap boundaries stop.
- **Owner:** client-view coordinator.

### WM-GO-010 — Persistent desktop lifecycle

- **Invariant:** create/update/reorder/switch preserve stable IDs and order; non-empty delete requires a
  different destination and transfers atomically; final desktop deletion is rejected.
- **Owner:** desktop reducer.

### WM-GO-011 — Focus desktop zoom

- **Invariant:** zoom creates one dedicated focus desktop; unchanged leaf, split, and stack sources
  restore their exact group/node IDs, order, weights, placement, and active member; edited sources
  use deterministic fallback; and every owner exit deletes an empty focus desktop or converts an
  occupied one to standard before commit.
- **Owner:** desktop/window reducer.

### WM-GO-012 — Undo and redo

- **Invariant:** complete operations round-trip topology, overrides, and revision semantics; new writes
  clear redo; history caps at configured length and never crosses workspace boundaries.
- **Owner:** history reducer/service.

### WM-GO-013 — Revision conflicts

- **Invariant:** stale expected revision rejects with current revision and structured conflicts; an
  unambiguous rebase is explicitly resolved before commit and ambiguous commands never write.
- **Owner:** service.

### WM-GO-014 — Workspace and client isolation

- **Invariant:** workspace A cannot list/read/mutate/stream B; client A and B in one workspace have
  independent active desktop/focus; remote presentation calls require an existing explicit client.
- **Owner:** service plus repository integration.

### WM-GO-015 — Repository reopen and malformed state

- **Invariant:** committed typed v2 state reopens exactly; unknown/v1/malformed documents are rejected
  without compatibility decoding; last-known-good state remains intact.
- **Owner:** bbolt repository adapter.

### WM-GO-016 — Configuration lifecycle

- **Invariant:** defaults and workspace overrides merge deterministically; invalid ranges, bindings, or
  shortcut conflicts reject the whole update and keep prior known-good config.
- **Owner:** config parser/service integration.

### WM-GO-017 — Declarative resources

- **Invariant:** valid `window_layout` resources validate/preview/apply through the same reducer;
  executable/unknown nodes, bad participants, and capability failures reject without writes.
- **Owner:** resource adapter plus service.

### WM-GO-018 — Subscription order and slow consumer

- **Invariant:** subscription snapshot fences events by revision, publications follow commit order,
  a client-bound fence plus presentation revision closes the register/subscribe race, remote
  switch/focus reaches only the selected client without topology revision/history/hook changes, and
  eviction of a slow consumer cannot block writers or other subscribers.
- **Owner:** service hub/repository integration.

### WM-GO-019 — Durable route intent

- **Invariant:** open/navigate/reopen/export/apply preserve canonical pathname/search;
  `window.navigate` performs one CAS/event, optionally focuses the explicit client, does not add or
  clear layout history, and layout undo/redo preserves the current route of surviving windows.
- **Owner:** window-manager service.
- **Canonical suite:** existing service contract, lifecycle, and layout-command suites.

### WM-GO-020 — JSON-safe revision exhaustion

- **Invariant:** a changed preview and every durable commit reject before incrementing a topology
  revision at the maximum JSON-safe value; repositories reject unsafe or wrapped commits and retain
  the prior snapshot/event state.
- **Owner:** window-manager service and repository contract.
- **Canonical suite:** existing service contract and repository suites.

## WM-TRANSPORT — Existing API/UDS/CLI/native suites

### WM-TRANSPORT-001 — HTTP/UDS parity

- **Invariant:** every endpoint returns equal status, body envelope, diagnostic code, workspace scope,
  revision, and validation behavior on HTTP and UDS.
- **Canonical suite:** existing API transport parity suite.

### WM-TRANSPORT-002 — Command schemas and status/body

- **Invariant:** malformed command/layout/client IDs, absent destinations, conflicts, authorization
  failures, and successful results map to stable OpenAPI bodies as well as correct status codes.
- **Canonical suite:** existing API contract/handler suite.

### WM-TRANSPORT-003 — CLI structured output

- **Invariant:** all desktop/window/layout verbs support structured output, explicit revision/client
  options, predictable exit codes, and the same error vocabulary.
- **Canonical suite:** existing CLI command suite.

### WM-TRANSPORT-004 — Native-tool parity

- **Invariant:** lazy toolset descriptors, capability gates, risks, schemas, digests, availability
  diagnostics, and execution results match the service/CLI/API contract.
- **Canonical suite:** existing native-tool registry/execution suite.

The transport matrix includes `window.navigate`, `agh window navigate --id --pathname
--search-json`, and native tool `agh__window_navigate`; malformed/non-object search JSON and external
or relative pathnames reject consistently on HTTP, UDS, CLI, and native-tool paths.

### WM-TRANSPORT-005 — Generated contract co-ship

- **Invariant:** OpenAPI, generated TypeScript, daemon E2E mock dispatch/matchers, and CLI reference are
  regenerated and compile together.
- **Owner:** `make codegen-check`, build, and existing E2E mock conformance gates.

### WM-TRANSPORT-006 — WebSocket frame and UDS parity

- **Invariant:** response 101 is the union of the four real stream frames; required wire arrays encode
  as `[]`, never `null`; HTTP and a real Unix-socket WebSocket both deliver the client-bound fence,
  terminal errors, and leak-free shutdown.
- **Owner:** existing API spec, core WebSocket, and UDS integration suites.

## WM-WEB-LIB — Consolidated OS projection/gesture suites

### WM-WEB-001 — Projection geometry

- **Invariant:** normalized split/stack/group plans project inside the measured work area with exact
  shared gaps, finite rects, no overlap, and deterministic rounding.
- **Canonical suite:** existing OS geometry/layout-engine suite, renamed around projection.

### WM-WEB-002 — Adaptive minimum-size stack

- **Invariant:** an impossible split becomes a temporary tabbed stack with all windows reachable and
  returns to split when space permits; stored topology is unchanged.
- **Canonical suite:** projection suite. This replaces the old overlap-freezing assertion.

### WM-WEB-003 — Shared seams

- **Invariant:** derived seam identity is `split_id + boundary_index`; keyboard/pointer resize moves all
  descendants on each side with real ARIA min/max values.
- **Canonical suite:** existing seam suite.

### WM-WEB-004 — Pointer containment and progressive targets

- **Invariant:** outside coordinates resolve no target; halves, quarters, thirds/two-thirds,
  before/after/side/stack, top-center zoom, and dock reservation honor bindings and hysteresis.
- **Canonical suite:** existing snap-zone/session suite.

### WM-WEB-005 — Gesture lifecycle and stale revisions

- **Invariant:** final mouse sample decides commit; Escape, pointercancel, lost capture, or ambiguous
  revision change cancels; unambiguous rebase sends one command; previews do not persist.
- **Canonical suite:** existing gesture hook suite.

### WM-WEB-006 — Floating clamp and drag-away

- **Invariant:** every commit remains in the work area with reachable title-bar grab point; drag-away
  detaches the window and modifier behavior moves the group.
- **Canonical suite:** existing drag/resize suite.

### WM-WEB-007 — Query/store reconciliation

- **Invariant:** snapshot fence plus events cannot regress revision, echo does not duplicate commands,
  disconnected state is explicit, and no partial multi-window state appears. TanStack Query remains the
  only server-snapshot authority; the domain Zustand store contains only shared client interaction and
  presentation state, exposes selector-scoped reads/grouped actions, and resets cleanly per workspace.
- **Canonical suite:** existing desktop persistence/store suite, hard-cut to window-manager contracts.

### WM-WEB-008 — Client-local desktop projection

- **Invariant:** switching one client locally or through CLI/native control changes only its
  active/inert desktop projection while all desktop application trees stay mounted and server
  topology remains shared; equal/stale presentation frames cannot regress the bound client view.
- **Canonical suite:** desktop store/provider suite.

## WM-WEB-UI — Component/story suites and visual evidence

### WM-UI-001 — Dot pager cardinalities

- **Invariant:** 1, 2, and 7 desktops show all; 8+ show active ±2 with accurate adaptive overflow
  counts; active indicator, buttons, and overview segment track client state.
- **Canonical suite:** desktop pager component suite and Storybook stories.

### WM-UI-002 — Pager accessibility

- **Invariant:** arrows, Home/End, Enter/Space, focus, `aria-current`, position/name, tooltips, and
  44px targets work without wrapping; reduced motion has no slide.
- **Canonical suite:** pager component suite; geometry/contrast through screenshot inspection.

### WM-UI-003 — Desktop overview

- **Invariant:** create/rename/reorder/delete-transfer, thumbnails, and drag-between-desktops call
  preview/command contracts and display loading/error/conflict states.
- **Canonical suite:** existing Spaces/overview component suite, renamed and repurposed.

### WM-UI-004 — Layout editor

- **Invariant:** declarative split/stack/weights/aspect/overflow documents validate and preview before
  apply; invalid documents display daemon diagnostics and cannot apply.
- **Canonical suite:** Settings window-management section suite.

### WM-UI-005 — Visual contract

- **Invariant:** pager is minimal lower-left horizontal chrome aligned with the Dock centerline,
  desktop transitions and preview overlays use canonical tokens, inactive desktops are
  visually/interaction inert, and Dock-safe placement holds.
- **Owner:** `eng-ui-screenshot` Storybook/reference/implementation bundle; no CSS-literal test.

## WM-E2E — Existing OS-shell browser lane

### WM-E2E-001 — Pointer arrange workflow

Drag a floating window into occupied side/corner/stack targets, inspect full preview, commit, resize a
one-to-many boundary, drag one child away, cancel another drag, and confirm reload persistence.

### WM-E2E-002 — Keyboard and command workflow

Use shortcuts/palette for focus, move, repeat ratios, arrange, balance, undo/redo, desktop switching,
and overview. Confirm focus does not wrap and announcements are truthful.

### WM-E2E-003 — Desktop persistence workflow

Create/reorder/rename desktops, move windows, reload, zoom through a dedicated focus desktop, restore
the exact source topology, delete a non-empty desktop with transfer, and verify mounted app state
survives switches.

Navigate an existing window to a detail route with search state, reload and restart the daemon, then
confirm the exact route resumes and converges in a second client without duplicating browser history.

### WM-E2E-004 — Multi-client workflow

Two isolated browser clients share topology but retain independent active desktop/focus. A remote CLI
or native command with explicit `client_id` changes only its target; missing/unknown client fails.

### WM-E2E-005 — Small viewport and reduced motion

Force impossible minima, confirm temporary stack and restoration, exercise pager at 8+ desktops,
confirm dock-safe placement, and verify reduced-motion crossfade/instant behavior.

### WM-E2E-006 — Raw recovery workflow

Export a layout, reject a malformed edit, validate a correct edit, apply with expected revision, then
observe one atomic update in both clients.

## Completion gates

- Focused Go race tests for `internal/windowmanager` and touched transport/tool/config packages.
- `bunx turbo run lint typecheck test --filter=./web` from repo root.
- Storybook capture through `eng-ui-screenshot`, with a valid story ID and inspected PNG.
- Content-addressed `untested` QA scenario files for the new user-visible behavior.
- One final `make verify` after the final source edit and after mandatory QA process teardown.
- `$deep-review` until `SHIP`, maximum three rounds; each confirmed finding is fixed in production
  rather than weakening tests.
