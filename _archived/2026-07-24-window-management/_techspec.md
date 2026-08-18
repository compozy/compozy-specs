# Technical Specification: Hybrid Window Management and Virtual Desktops

## Executive Summary

This program replaces the OS shell's client-owned independent snap fractions with a typed,
daemon-authoritative hybrid window manager. An AGH workspace owns persistent virtual desktops;
desktops own non-overlapping tiled groups plus floating windows; groups own normalized split, stack,
and leaf topology. The browser remains responsible for projecting normalized intent into its measured
work area and adapting an impossible minimum-size split into a temporary stack.

There is no separate PRD for this hardening program. The accepted product decisions in
`.codex/plans/2026-07-22T225426-0300-window-management.md`, the three analyses in this task directory,
and ADR-001 are the approved requirements. The old public `desktop-state` API is deleted rather than
adapted. HTTP, UDS, CLI, native tools, pointer gestures, shortcuts, the command palette, and the
declarative layout editor all invoke the same semantic service.

Primary trade-offs are a broad public-contract hard cut, a new typed daemon domain, and client/server
conformance fixtures. In return, group boundaries, atomic arrangements, virtual desktops, undo/redo,
multi-client revision safety, configuration, and agent manageability become explicit invariants.

## MVP Boundary

**MVP boundary: everything in this specification ships in this program.** This includes persistent
desktops, tiled groups plus floating windows, split/stack topology, shared-boundary resizing,
progressive snap targets, structural occupied drops, minimum-size stack adaptation, directional focus,
close/minimize/restore reflow, focus-desktop zoom, transactional undo/redo, the lower-left horizontal
dot pager aligned with the Dock centerline,
on-demand desktop overview, global defaults plus workspace overrides, a declarative layout editor,
HTTP/UDS/CLI/native-tool parity, declarative `window_layout` resources, official-skill/docs updates,
and migration-free deletion of the v1 public contract.

Explicitly out of scope: physical browser/OS windows, physical displays, macOS Accessibility APIs,
pin-to-all-desktops, automatic wraparound desktop navigation, executable custom geometry, implicit
foreground-client selection, and compatibility readers or aliases for v1 state.

## Delete Targets

Delete or replace these surfaces in the same program; do not retain aliases or dual writes:

- `web/src/systems/os/lib/os-types.ts`: `OsSnapZone` and durable `snap`, `maximized`, `z`, and
  `prevRect` semantics.
- `web/src/systems/os/lib/os-snap-seams.ts`: pairwise seam derivation.
- `web/src/systems/os/stores/desktop-store.ts`: `snapWindow`, `resizeSnapped`, `splitWindows`, and
  `arrangeWindows` sequential mutation paths.
- `web/src/systems/os/lib/desktop-persistence.ts` and payload codecs: v1 `desktop`/`win:*` public
  documents and compatibility decoding.
- `internal/api/contract/desktopstate.go`, desktop-state OpenAPI paths, handlers, CLI verbs, generated
  clients, CLI docs, and `skills/agh/references/desktop-state.md`.
- Workspace-facing `Space` identifiers and labels in the OS system. The hard-cut nouns are
  `Workspace` (AGH isolation boundary) and `Desktop` (persistent virtual arrangement).
- ADR-009 sections in `.compozy/tasks/os-shell/` that define independent snap fractions; add a
  superseding amendment/reference to this TechSpec rather than leaving two active contracts.

The internal `clientstate` bbolt engine may be reused as persistence infrastructure only. No old
public handler, DTO, v1 decoder, fallback read, or raw domain write survives.

## System Architecture

### Component overview

**Daemon:**

- `internal/windowmanager`: contract types, reducer, normalization, validation, history, repository,
  event subscriptions, layout resources, and the public service.
- `internal/api/contract`: canonical command, snapshot, preview, layout, client, and event schemas.
- `internal/api`: workspace-authorized HTTP/UDS handlers and stream.
- `internal/cli`: structured `desktop`, `window`, and `layout` command groups.
- native tools: lazy `agh__window_manager` toolset using the same service/fallback contracts.
- configuration: validated global `[window_manager]` defaults with typed workspace overrides.

**Web (`web/src/systems/os`):**

- `adapters/window-manager-api.ts`: generated-contract wrapper and typed error.
- `lib/window-manager-schemas.ts`: Zod parsing at the network boundary.
- `lib/layout-projection.ts`: pure normalized-to-pixel projection and adaptive stack decisions.
- `lib/snap-targets.ts`: pure pointer-intent resolver and progressive target catalog.
- `stores/window-manager-store.ts`: domain-focused Zustand state for optimistic preview, gesture
  sessions, overview/editor visibility, local transition intent, and command coordination; durable
  server snapshots and revisions stay in the canonical query cache.
- `hooks`: stream reconciliation, gestures, shortcuts, swipe, and route coordination.
- `components`: desktop layer, tiled groups, floating frames, command previews, minimal dot pager,
  desktop overview, and declarative layout editor.

### Architectural Boundaries

- `internal/windowmanager` owns domain types and interfaces. It imports no API, CLI, tool, web, or
  daemon package. Persistence adapters implement interfaces owned by `windowmanager`.
- `internal/api`, `internal/cli`, and native-tool adapters translate public DTOs to domain commands;
  they never implement mutation logic or normalization.
- The daemon composition root wires the service, config, workspace authorizer, resource registry,
  and store. It contains no command behavior.
- The web dependency flow is `adapters -> lib -> hooks/stores -> components`; cross-system imports use
  public barrels only. Generic visual primitives remain in `@agh/ui`.
- TanStack Query is the sole authority for server snapshots and revisions. Zustand contains complex
  shared browser presentation/gesture state and references the current snapshot; it does not create a
  second durable server-state cache. The store exposes narrow typed selectors and grouped actions.
  Shell/domain components consume those selectors directly instead of prop-drilling. Derived state is
  computed synchronously or by store/query subscriptions, not synchronized through `useEffect` chains;
  leaf visual components may still accept local render props.
- Layout resource providers contribute declarative documents to a registry. They never receive
  pointer events, arbitrary execution, persistence handles, or mutable service internals.
- `workspace_id` is resolved and authorized before every read, preview, mutation, stream, CLI call,
  native-tool call, or resource application. `client_id` is an additional key, never a substitute.

### Mutation pipeline

Every durable command runs inside one workspace serialization boundary:

1. Read and authorize workspace plus the expected revision.
2. Clone the snapshot and resolve explicit targets.
3. Reduce exactly one semantic command.
4. Normalize topology deterministically.
5. Validate all safety and workspace/client invariants.
6. Commit snapshot, revision, history record, and event as one repository transaction.
7. Publish only after the commit succeeds.

Preview executes steps 1–5 and returns a proposed snapshot/diff/diagnostics without step 6 or 7.
Undo and redo are semantic commands over history and obey the same pipeline.

## Implementation Design

### Core Go interfaces

```go
package windowmanager

type WorkspaceID string
type DesktopID string
type WindowID string
type NodeID string
type GroupID string
type ClientID string
type Revision uint64

type Service interface {
    Snapshot(ctx context.Context, workspaceID WorkspaceID) (Snapshot, error)
    Preview(ctx context.Context, request CommandRequest) (Preview, error)
    Execute(ctx context.Context, request CommandRequest) (Result, error)
    Clients(ctx context.Context, workspaceID WorkspaceID) ([]ClientView, error)
    RegisterClient(ctx context.Context, registration ClientRegistration) (ClientView, error)
    UnregisterClient(ctx context.Context, workspaceID WorkspaceID, clientID ClientID) error
    ExportLayout(ctx context.Context, workspaceID WorkspaceID) (LayoutDocument, error)
    ValidateLayout(ctx context.Context, workspaceID WorkspaceID, document LayoutDocument) (Validation, error)
    ReplaceLayout(ctx context.Context, request ReplaceLayoutRequest) (Result, error)
    Subscribe(ctx context.Context, workspaceID WorkspaceID, after Revision) (Subscription, error)
}

type Repository interface {
    Load(ctx context.Context, workspaceID WorkspaceID) (Snapshot, error)
    Commit(ctx context.Context, commit Commit) error
    DeleteWorkspace(ctx context.Context, workspaceID WorkspaceID) error
}

type WorkspaceResolver interface {
    ResolveWorkspace(ctx context.Context, workspaceID WorkspaceID) error
}

type LayoutResourceRegistry interface {
    Resolve(ctx context.Context, workspaceID WorkspaceID, resourceID string) (LayoutDocument, error)
    List(ctx context.Context, workspaceID WorkspaceID) ([]LayoutResource, error)
}

type Subscription interface {
    Snapshot() Snapshot
    Events() <-chan Event
    Err() error
    Close() error
}
```

The concrete service constructor is:

```go
func NewService(
    repository Repository,
    workspaces WorkspaceResolver,
    layouts LayoutResourceRegistry,
    defaults Config,
    options ...Option,
) (*Manager, error)
```

All errors are wrapped with `%w` and mapped deterministically: `ErrWorkspaceNotFound`,
`ErrSnapshotNotFound`, `ErrRevisionConflict`, `ErrInvalidCommand`, `ErrInvalidTopology`,
`ErrDesktopNotFound`, `ErrWindowNotFound`, `ErrClientNotFound`, `ErrDestinationRequired`,
`ErrFinalDesktop`, `ErrHistoryBoundary`, `ErrLayoutResourceNotFound`, and `ErrSlowConsumer`.

### Commands

`CommandRequest` contains `workspace_id`, `command_id`, `expected_revision`, optional explicit
`client_id`, actor/origin metadata, and a typed command payload. Stable IDs are:

- `desktop.create`, `desktop.update`, `desktop.reorder`, `desktop.switch`, `desktop.delete`
- `window.open`, `window.close`, `window.focus`, `window.move`, `window.swap`,
  `window.toggle_floating`, `window.zoom`, `window.navigate`
- `layout.arrange`, `layout.resize`, `layout.balance`, `layout.undo`, `layout.redo`,
  `layout.replace`

Remote `desktop.switch`, `window.focus`, and `window.zoom` require a connected `client_id`.
`window.navigate` updates durable route intent without a client; with an explicit connected
`client_id`, the same successful commit also activates the window's desktop and focuses it for that
client. Browser requests always include their registered stable local client ID. Agent `arrange`
requests include explicit window IDs and never infer participants from a UI selection.

### Data Models

```go
type Snapshot struct {
    Version       uint32
    WorkspaceID   WorkspaceID
    Revision      Revision
    Desktops      []Desktop
    Windows       map[WindowID]Window
    History       History
    Overrides     WorkspaceConfig
    UpdatedAt     time.Time
}

type Desktop struct {
    ID          DesktopID
    Name        string
    Order       int
    Purpose     DesktopPurpose // standard | focus
    FocusOwner  *WindowID
    Groups      []LayoutGroup
    Floating    []WindowID
}

type LayoutGroup struct {
    ID    GroupID
    Frame NormalizedRect
    Root  LayoutNode
}

type LayoutNode struct {
    ID       NodeID
    Kind     NodeKind // leaf | split | stack
    WindowID *WindowID
    Axis     *Axis
    Children []LayoutNode
    Weights  []float64
    WindowIDs []WindowID
    ActiveID *WindowID
}

type Window struct {
    ID           WindowID
    App          string
    InstanceKey  *string
    Placement    WindowPlacement
    Route        RouteIntent
    DesktopID    DesktopID
    FloatingRect NormalizedRect
    Minimized    bool
    ReturnAnchor *ReturnAnchor
}

type RouteIntent struct {
    Pathname string
    Search   json.RawMessage // canonical JSON object
}

type ReturnAnchor struct {
    DesktopID       DesktopID
    GroupID         *GroupID
    ParentSplitID   *NodeID
    Axis            *Axis
    ChildIndex      *int
    Weight          *float64
    NeighborIDs     []WindowID
    SourceRevision  Revision
    SourceGroup     *LayoutGroup
}

type ClientView struct {
    WorkspaceID     WorkspaceID
    ClientID        ClientID
    PresentationRevision uint64
    ActiveDesktopID DesktopID
    FocusedWindowID *WindowID
    FocusOrder      []WindowID
    ConnectedAt     time.Time
}
```

Field rationale:

| Field | Rationale |
|---|---|
| `Version` | Rejects unknown documents; no v1 decoder or migration path exists. |
| `Revision` | Provides command CAS and stale-gesture protection across clients. |
| ordered `Desktops` plus `Order` | Stable navigation and deterministic repair/export; validator requires one contiguous ordering. |
| `Purpose`/`FocusOwner` | Makes an active dedicated focus desktop persistent and inspectable without hidden naming conventions. |
| group `Frame` | Allows multiple independent tiled islands without forcing automatic full-screen tiling. |
| n-ary `Children`/`Weights` | Represents a boundary with every descendant on each side and normalizes same-axis nesting. |
| stack `WindowIDs`/`ActiveID` | Provides explicit tab order and focus when tiling is impossible or intentionally stacked. |
| `FloatingRect` | Preserves normalized manual geometry across viewport changes; pixels never enter durable state. |
| `Placement` | Separates tiled/stacked/floating topology from application navigation. |
| `Route` | Restores canonical pathname/search after reload or restart and converges across peer clients. |
| `ReturnAnchor` | Carries the structural slot plus a validated deep `source_group`. Zoom restores that exact group identity, node order, weights, and active stack member only while the source residue is unchanged; later edits use deterministic fallback. |
| `ClientView` | Separates shared topology from per-client active desktop/focus; its own revision fences remote presentation updates without pretending they are topology writes. |
| `History` | Stores operation-level before/after documents for exact workspace-scoped undo/redo, bounded by config. |

### Storage-shape decision: side-table vs JSON

No SQLite migration or relational side table is introduced. Window-manager state is a single typed,
versioned workspace document in the existing bbolt client-state engine because the daemon always
loads, validates, and commits the complete topology as one aggregate; it does not filter, join, or
page individual nodes. The public service parses the document into domain structs before use, so the
payload is not public opaque JSON. A bbolt repository key is `window_manager/snapshot`; commit
metadata and event sequence stay in the existing engine transaction.

Client views are transient connection state keyed by `(workspace_id, client_id)` and are not stored in
SQLite or the shared snapshot. A browser persists only its stable random client identifier locally.
If future requirements need server-side queries across individual windows, that is a new schema design
with an append-only migration; this program must not add partial JSON indexes or dual storage.

### Normalization

Normalization is pure and idempotent:

1. Remove missing leaf references and duplicates after their first deterministic occurrence.
2. Remove empty nodes and groups.
3. Collapse a non-root single-child split; convert one-window stacks to leaves.
4. Flatten nested splits with the same axis while multiplying parent/child weights.
5. Replace non-finite/non-positive weights with equal shares and normalize the sum to one.
6. Remove tiled windows from floating order and de-duplicate floating order.
7. Rebuild contiguous desktop order and deterministic focus fallback references.

Normalization repairs only unambiguous representational residue caused by a valid command. Imported
documents with cycles, cross-desktop duplicates, overlapping group frames, unknown window members, or
ambiguous ownership are rejected rather than salvaged.

### Safety Invariants

1. Every window belongs to exactly one desktop and is exactly one of tiled, stacked, or floating.
2. A layout node ID and window leaf occur at most once within a workspace snapshot.
3. Group frames on one desktop are finite, normalized, positive, in bounds, and non-overlapping.
4. Split child and weight counts match; weights are finite, positive, and sum to one within tolerance.
5. The final committed topology has no cycles, empty containers, or non-root singleton containers.
6. A successful command advances `Revision` exactly once and persists one snapshot and semantic
   event in one transaction. Layout/topology commands add one history entry; `window.navigate` never
   adds or clears layout history.
7. Preview, validation, no-op, and rejected commands make no durable write and publish no event.
8. Undo and redo restore complete topology/config overrides atomically and never cross workspaces.
9. A stale expected revision returns the current revision plus structured conflict details. Pointer
   releases rebase only if source and target node identities are unchanged and resolution is unique.
10. Workspace authorization is performed before store/cache/stream access; all keys include
    `workspace_id`, and event fan-out never uses client-supplied workspace data after authorization.
11. Client-local mutations require a live `(workspace_id, client_id)` registration. Remote callers
    cannot select an implicit foreground client.
12. Deleting a non-empty desktop requires a distinct explicit destination and transfers every
    window/group or rejects the entire command.
13. The last desktop cannot be deleted. A workspace always normalizes to at least one standard desktop.
14. Event publication follows successful commit order; a slow subscriber is evicted and cannot block
    the workspace command sequencer.
15. Configuration and layout-resource changes become active only after complete parse and validation;
    the prior known-good configuration remains active on failure.
16. Every window carries an absolute internal pathname and a canonical JSON-object search value.
    Layout undo/redo and replacement preserve the current route of surviving window IDs.
17. A stream bound to a registered `client_id` fences that client's current view and emits only its
    later monotonically revisioned presentation frames. Remote focus/switch never advances topology
    revision/history/hooks, and another client never receives the frame.

### Window behavior

- New windows open floating on the invoking client's active desktop. A validated workspace override
  may insert beside focus instead.
- Closing or minimizing a grouped window removes its leaf and reflows the group. Restore uses the
  return anchor when the parent/neighbor relationship still exists; otherwise it inserts beside the
  client's focused window, then falls back to floating.
- Zoom moves one window to a dedicated focus desktop. Its return anchor captures the exact source
  group. Restore reinstalls that group when the remaining live residue still matches; source edits
  use the structural-anchor fallback. Whenever the owner leaves through restore, move, swap,
  minimize, close, or layout replacement, the daemon deletes an empty focus desktop or converts an
  occupied one to standard. A later zoom creates a new focus desktop adjacent to the source.
- Drag-away detaches the selected window by default. The configured modifier moves the whole group.
- An occupied drop resolves to before/after, orthogonal side, or center stack and returns a full
  structural preview.
- `layout.resize` targets `split_id + boundary_index`, adjusting the two adjacent weights and every
  projected descendant. `balance` equalizes the selected split or all splits in a group.
- Repeat snapping cycles ratios `0.5`, `0.666667`, `0.333333` only when the current structural target
  still matches the previous action.
- Opening a window commits its canonical route. `window.navigate` updates that route by revision/CAS;
  equal routes are no-ops, and navigation does not become part of layout undo/redo.

### Web projection contract

The projection input is snapshot revision, one desktop, measured work-area pixels, configured gaps,
aspect class, per-window minimums, and client focus. It returns rects, derived seams, stack adaptations,
and diagnostics. Outer gaps are applied at group boundaries; inner gaps are shared once. A branch whose
minimum sum exceeds available pixels returns a temporary stack projection. Stored topology is unchanged
and automatically returns when the viewport permits.

Floating commits clamp to the work area and keep the grabbed title-bar point reachable. Snap intent is
`none` outside the work area. Gesture sessions capture revision and final pointer coordinates and cancel
on Escape, pointer cancellation, lost capture, invalid client registration, or ambiguous stale state.

### Desktop UI

The active client's desktop pager occupies the lower-left segment of the same bottom-chrome row as the
Dock, with both controls sharing one vertical centerline. It has no resting card, label, border, or
background. Inactive dots are `6x6`; the active indicator is a `14x6` pill; each control retains an
invisible `44x44` hit target. Up to seven
desktops show every indicator. Beyond seven, show active plus two neighbors on each side and adaptive
earlier/later overflow controls. Overflow announces hidden counts and opens the on-demand overview at
that segment.

The pager supports click, arrow keys, Home/End, Enter/Space, `aria-current`, names, positions, tooltips,
reduced motion, and client-specific updates. Desktop transitions slide horizontally and stop at ends;
reduced motion uses an instant/crossfade transition. All desktops stay mounted so application state is
preserved, while inactive desktop trees are inert, non-interactive, accessibility-hidden, and rendering
contained.

The `Desktops Overview` owns create, rename, reorder, thumbnails, drag-between-desktops, and explicit
transfer-before-delete. Existing workspace overview copy becomes `Workspaces` and does not share the
desktop state model.

### Config Lifecycle

Global `config.toml` defaults and workspace overrides use the same schema:

```toml
[window_manager]
new_window_policy = "floating"
small_viewport_policy = "stack"
focus_policy = "click_directional"
focus_wrap = false
drag_away_policy = "window"
history_limit = 50
desktop_transition = "slide"

[window_manager.gaps]
inner = 8
top = 8
right = 10
bottom = 8
left = 10

[window_manager.snap]
edge_band = 32
corner_reach = 150
exit_slack = 16
repeat_ratios = [0.5, 0.666667, 0.333333]
```

The complete schema also covers focus-follows-pointer, raise-on-focus, shortcuts, the group-move
modifier, and remapping top-center/bottom-center to `zoom`, `reserved`, or `none`. Portrait/landscape
profiles use `window_layout.aspect_variant`, not a second config authority. Parsing validates ranges,
duplicates, shortcut conflicts, target IDs, ratio bounds, and impossible policies. Global reload and
workspace override writes swap atomically after full validation and publish one durable config event.
The global Settings surface is `GET/PATCH /api/settings/window-manager`; every key is live-classified,
and a failed write/apply retains the prior active generation.

### Extensibility Integration Plan

The extension impact is deliberate and bounded. Extension manifests and bundles may advertise the
existing resource capability plus the new `window_layout` kind. The resource registry validates each
document before discovery; bridge SDKs and MCP sidecars expose resource metadata but cannot execute
geometry or subscribe to pointer/focus churn. Skills/capabilities gate discovery and apply separately.
Protocol documentation defines the resource version and stable diagnostic codes. Durable hooks are
emitted only after a committed layout apply, desktop create/delete, or window move. Existing tool and
resource registries are extended; no parallel registry is introduced.

#### Declarative layout resources

`window_layout` resources contain version, ID, display name, optional aspect variant, a declarative
split/stack template, default weights, participant slots, and overflow policy. The Settings editor
creates the same document and calls validate/preview before apply. Extension registries, bundles,
bridge SDKs, and MCP sidecars may contribute resources through the existing resource registry and
capability gates. They cannot contribute executable code. Durable hooks cover layout applied, desktop
created/deleted, and window moved; pointer previews and client focus churn do not emit extension hooks.

## Public Control Surface

HTTP and UDS expose identical contracts:

- `GET /api/workspaces/{workspace_id}/window-manager`
- `POST /api/workspaces/{workspace_id}/window-manager/preview`
- `POST /api/workspaces/{workspace_id}/window-manager/commands`
- `GET /api/workspaces/{workspace_id}/window-manager/clients`
- `POST /api/workspaces/{workspace_id}/window-manager/clients`
- `DELETE /api/workspaces/{workspace_id}/window-manager/clients/{client_id}`
- `GET /api/workspaces/{workspace_id}/window-manager/layout`
- `POST /api/workspaces/{workspace_id}/window-manager/layout/validate`
- `PUT /api/workspaces/{workspace_id}/window-manager/layout`
- `GET /api/workspaces/{workspace_id}/window-manager/stream?client_id={client_id}`
- `GET /api/settings/window-manager`
- `PATCH /api/settings/window-manager`

The stream's initial snapshot frame includes the bound current `ClientView`; later `client` frames
carry its presentation revision independently from topology revision. Omitting `client_id` creates a
topology-only agent subscription. Every mutation returns revision, changed IDs, diagnostics, and preview/apply parity metadata. OpenAPI is
the canonical schema; generated TypeScript, E2E mock matchers, and CLI reference ship in the same
change.

CLI groups:

- `agh desktop list|create|update|reorder|switch|delete|clients`
- `agh window list|open|close|focus|move|swap|float|zoom|navigate`
- `agh layout get|preview|arrange|resize|balance|undo|redo|export|validate|apply`

Structured JSON is available for every verb. Presentation mutations require `--client`; mutation
verbs accept `--revision`; arrange/move require explicit IDs. Human output never changes the contract.

### Agent Manageability Plan

The lazy `agh__window_manager` toolset provides 26 matching semantic tools, including
`agh__window_navigate`, and layout
export/validate/apply. Descriptors declare workspace capability requirements, mutation/read risk,
approval behavior, schemas, and availability diagnostics. Native tools use the in-process service;
CLI/API fallbacks use the same generated contract and error vocabulary.

## Web/Docs Impact

- Web routes/components/hooks: `web/src/systems/os` replaces snap types, store actions, gestures,
  seams, persistence, Spaces overlay, command registry, and Settings integration. TanStack Query owns
  topology, durable route intent, and daemon ClientView query state; Zustand holds only pending
  navigation/gesture intent while mutations are in flight, and route shell mounting consumes the
  focused window's snapshot route.
- Shared UI: first inspect `packages/ui/src/index.ts`; only genuinely generic pager/layout-editor
  primitives may land in `packages/ui` with stories/tests. Domain composites remain in `systems/os`.
- Docs: replace desktop-state CLI/API documentation, update configuration reference, OS-shell user
  guide, generated OpenAPI/CLI refs, and `skills/agh/` semantic workflows.
- Copy: public nouns are Workspace and Desktop. Copy follows `COPY.md` and the glossary; no surface
  claims physical OS window management.

## Assumptions and Defaults

- `UNCONFIRMED` is empty: all behavior branches were resolved in the accepted grill.
- The existing workspace service remains the authorization and lifecycle owner.
- The existing bbolt engine can provide one-key compare-and-swap/transactional commit and ordered
  events; if inspection disproves this, the repository adapter must extend internal mechanics without
  restoring public desktop-state access.
- Browser local storage may hold the random stable `client_id` only, never workspace topology.
- Layout history is workspace-scoped and limited to 50 by default.
- Desktop navigation does not wrap. A window is never pinned to all desktops.

## Test Plan and Ownership

The detailed catalog is `_tests.md`. Placement decisions:

- Domain invariants, normalization, commands, history, atomicity, and workspace isolation belong to
  the canonical `internal/windowmanager` package suite.
- HTTP/UDS DTO/status/body behavior extends the existing API transport suites; CLI output/exit behavior
  extends existing CLI suites; native-tool descriptors/execution extend native-tool suites.
- Projection, targets, seams, and gesture lifecycle extend consolidated OS lib/hook Vitest suites.
- Snapshot/query reconciliation extends the existing desktop persistence/store suite rather than a
  duplicate standalone suite.
- Pager/layout editor presentation and interactions use component/story suites; pixels are verified by
  Storybook capture, not CSS-literal or snapshot assertions.
- Real pointer, keyboard, reload, multi-client, and desktop workflows extend the OS-shell E2E lane.

Tests never assert obsolete type names, file existence, generated output text, or exact CSS literals.
The former minimum-size overlap expectation is replaced because it exposed a production bug; it is not
weakened.

## Rollout and Recovery

This is a greenfield hard cut with no state migration. On first use, the service creates one standard
desktop and opens windows from route intent. Unknown or v1 documents are rejected and ignored; there is
no decode fallback. A malformed new document prevents mutation, returns diagnostics, and preserves the
last known-good committed snapshot. Raw layout export is the recovery artifact; validate plus apply is
the only replacement path.

## AGH Impact Audit

- **Native tools:** add lazy `agh__window_manager` desktop/window/layout descriptors, schemas, digest
  participation, mutation risks/approvals, capability gates, diagnostics, execution tests, and CLI/API
  fallbacks; remove any guidance that treats raw desktop-state CRUD as a management tool.
- **Extensibility and hooks:** add declarative `window_layout` resource registration/validation for
  extensions, bundles, bridge SDKs, and MCP sidecars; add durable topology lifecycle hooks; reuse the
  resource/capability registries; add complete config lifecycle. Pointer previews and client focus emit
  no hook.
- **Workspace data isolation:** snapshots, revisions, groups, windows, desktops, history, overrides,
  store keys, API/UDS/CLI/tool reads, stream subscriptions, events, and caches are workspace-scoped.
  Active desktop/focus additionally use explicit `client_id`; tests prove two workspaces and two clients
  cannot observe or mutate each other's state.
- **Official AGH skill:** replace desktop-state documentation with semantic desktop/window/layout
  commands, explicit client targeting, raw export/validate/apply recovery, declarative resources,
  configuration, and conflict diagnostics.

## References

- `.compozy/tasks/window-management/analysis/01_analysis_current-system.md`
- `.compozy/tasks/window-management/analysis/02_analysis_reference-topology.md`
- `.compozy/tasks/window-management/analysis/03_analysis_reference-interactions.md`
- `.compozy/tasks/window-management/adrs/adr-001.md`
- `.resources/amethyst/Amethyst/Layout/Layout.swift`
- `.resources/amethyst/Amethyst/Layout/Layouts/BinarySpacePartitioningLayout.swift`
- `.resources/arepospace/Sources/AppBundle/tree/TreeNode.swift`
- `.resources/arepospace/Sources/AppBundle/tree/normalizeContainers.swift`
- `.resources/arepospace/Sources/AppBundle/layout/layoutRecursive.swift`
- `.resources/rectangle/Rectangle/Snapping/SnappingManager.swift`
- `.resources/rectangle/Rectangle/WindowManager.swift`
- `.resources/rectangle/Rectangle/CooperativeCornerResize.swift`
- `https://herdr.dev/docs/socket-api/`
