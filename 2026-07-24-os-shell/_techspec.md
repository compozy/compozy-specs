# Technical Specification: AGH OS Shell (`os-shell`)

## Executive Summary

This spec turns the web SPA's chrome into the AGH OS v2 desktop: a menubar, a wallpapered desktop with free-floating windows, a dock, and a sessions rail — while keeping every existing route view. The refactor exploits the app's existing layering (thin route files → router glue → pure `systems/*` views): views move into window bodies unchanged; only the glue layer is rewritten as per-window controllers. Windows are app-scoped with internal navigation; the URL always reflects the focused window; TanStack Router stays the single navigation authority (no parallel routes exist upstream). Desktop state persists daemon-side in a new **desktop-state service** (internal package `clientstate`) — bbolt-backed KV with a per-workspace commit sequence and real-time WebSocket sync — giving multi-client convergence and agent-manageable surfaces without SQLite migration ceremony for high-churn presentation payloads (ADR-008).

The program's spec set is: `_prd.md` + `_user_stories.md` (US-001..021, product layer), this TechSpec, `_tests.md`, ADR-001..009, the dated market research at `analysis/01_analysis_market-research.md`, the Hermes window-arrangement analysis at `analysis/02_analysis_hermes-window-arrangement.md`, and the prototype (`docs/design/opendesign/os/agh-os-v2.html` + `os-v2.js`/`os-v2.css`/`os-v2-apps.css` + `OS-V2-DELIVERY.md`) — whose current working-tree contents, **including the complete `<960px` compact mode**, are the visual reference. Primary trade-offs: a second (deliberately narrow) storage engine in the daemon, one new frontend dependency (`react-rnd`), and full-parity MVP scope. Peer-review round 1 findings are incorporated (see `qa/peer-review-incorporation-round1.md`); the public persistence surface is named **desktop state** (ADR-008).

## MVP Boundary

**MVP boundary: everything in this spec ships in v1** — shell (menubar, desktop, dock, sessions rail), window manager (open/close/focus/z/drag/resize/minimize/zoom), windows for all 14 apps with internal navigation, multi-instance session windows, URL↔WM sync via the routing coordinator, the daemon desktop-state service (bbolt + WS + HTTP/UDS + CLI), truthful attention surfaces (dock badges, sessions rail, menubar bell aggregator), global ⌘K palette (RuntimeSelector → ⌘J), spaces (per-workspace arrangements + ⇧⌘S overview), wallpapers + Appearance pane, dock magnification, the **window snap layer** (ADR-009 — halves/quarters via drag-to-edge zones, palette, and keyboard, persisted as viewport-proportional fractions), and the compact (<960px) presentation mode implemented against the prototype's existing compact reference (Visual Contract Mode). Explicitly **post-MVP** (named follow-ups, out of scope here): tiling window mode (Hermes-style zone tree with tabbed zones), snap-layout editor / custom zones / multi-zone span / linked seam resize, auto-arrange ("tidy"), dock reordering (`@dnd-kit`), popout-to-native-browser-window, any second desktop-state consumer or extension-facing state API (ADR-008), native `agh__*` desktop-arrangement tools, inline approve/deny inside the bell popover (v1 bell focuses the owning window), and NATS-style pub/sub infrastructure.

## Delete Targets

Hard cuts in the same program — no aliases, no dual paths (SD-002, L-006). This is the executable per-file inventory (peer-review B-011); each port task deletes its rows in the same change.

**DELETE — shell chrome & ⌘K registry:**
`web/src/routes/_app/-app-shell.tsx` · `web/src/components/topbar-shell.tsx` · `web/src/hooks/routes/use-topbar-shell-model.ts` · `web/src/hooks/routes/use-app-layout.ts` (desktop shell hook replaces it; its stream mounts move to `DesktopShell`) · `web/src/systems/runtime/components/app-sidebar.tsx` · `web/src/stores/sidebar-store.ts` + `web/src/hooks/use-sidebar-store.ts` · `web/src/systems/runtime/components/runtime-selector/command-k-registry.ts` + `use-command-k-ownership.ts` (ADR-005) · every `data-testid="app-shell"/"app-grid"` assertion (updated by the owning task).

**DELETE — per-app route glue in `web/src/routes/_app/` (replaced by window controllers in `systems/os/apps/`):**
`-home-page.tsx` · `-tasks-route.tsx` · `-tasks-new-route.tsx` · `-tasks-edit-route.tsx` · `-agent-detail-page.tsx` · `-session-page.tsx` · `-sandbox-page.tsx` · `-vault-page.tsx` · `-settings-shell.tsx` · `-settings-shell-frame.tsx` · `-settings-shell-state.tsx` · `-settings-shell-boundaries.tsx` · `-settings-section-nav.tsx`.

**DELETE — per-page view-model hooks in `web/src/hooks/routes/` (each rewritten as its app's window controller, then removed):**
`use-home-page.ts` · `use-tasks-*`/task route state (`use-task-create-route-state.ts`, `use-task-detail-orchestration-tab.ts` if route-bound after port review) · `use-agents-fleet-page.ts` · `use-agent-detail-page.ts` · `use-session-detail-page.ts` · `use-automation-page.ts` + `use-automation-page-base.ts` + `use-automation-jobs-page.ts` + `use-automation-job-detail-page.ts` + `use-automation-triggers-page.ts` + `use-automation-trigger-detail-page.ts` · `use-bridges-page.ts` + `use-bridge-detail-page.ts` · `use-knowledge-page.ts` · `use-loops-catalog.ts` + `use-loop-detail.ts` + `use-loop-run-page.ts` + `use-loop-runs-route.ts` · `use-sandbox-page.ts` · `use-settings-page.ts` + the nine `use-settings-*-page.ts` files.

**REWRITE (new owner, old file deleted in the same change):**
`-app-route-boundaries.tsx` + `-app-route-boundary-frame.tsx` → per-window error boundaries in `systems/os` · `-onboarding-gate-frame.tsx` → desktop-level onboarding gate in `DesktopShell` · `-sandbox-dialogs.tsx` → `systems/sandbox` components · `use-session-topbar-slot.tsx` → per-window topbar slot publication · `use-mcp-authorize.ts` → window-scoped authorize flow controller.

**KEEP (reused as-is):**
all preload modules (`-app-preload.ts`, `-route-preload.ts`, `-agents-preload.ts`, `-automation-preload.ts`, `-bridges-preload.ts`, `-knowledge-preload.ts`, `-loops-preload.ts`, `-network-preload.ts`, `-settings-preload.ts`, `-tasks-preload.ts`, `-vault-preload.ts`) · `-agent-session-route-loader.ts` · `-session-permalink-route.ts` (redirect) · pure helpers (`loop-bindings-map.ts`, `task-catalog-route-filter.ts`) · domain action hooks (`use-session-clear-dialog.ts`, `use-session-delete-dialog.ts`, `use-session-workspace-guard.ts`, `use-session-page-controls.ts` content absorbed by the session window controller, `use-bridge-create-flow.ts`, `use-bridge-setup-flow.ts`, `use-bridge-delivery-tests.ts`, `use-loop-bindings.ts`, `use-create-provider-focus-restore.ts`) · route files (become sync-controllers) · `active-workspace-store` · `/design-system` standalone tree.

Spec-artifact sweep in the same program: stale "no PRD/stories" wording in this spec set (removed in round-1 incorporation) and any `.compozy/tasks/os-shell/*` references superseded by renames.

## System Architecture

### Component Overview

**Daemon (Go):**

- `internal/clientstate` — new package: `Service` (CRUD + watch + purge), bbolt store, and the fan-out hub. Owns `$AGH_HOME/state/clientstate.db`. Knows nothing about the OS shell; payloads are opaque JSON under `(workspace, domain, key)`.
- `internal/api` — HTTP/UDS handlers for desktop-state CRUD/apply + the WebSocket stream endpoint on both listeners (`gorilla/websocket`, already a dependency); canonical DTO + OpenAPI schemas; deterministic error codes.
- `internal/cli` — `agh desktop-state` verb family with `-o json`/`-o jsonl` structured output.
- `internal/daemon` — composition root wiring only (SD-008): constructs the service, registers routes, hooks workspace deletion → `PurgeWorkspace`.

**Web (`web/src/systems/os/` — new system):**

- `stores/desktop-store.ts` — Zustand WM store: windows, focus, z, presentation mode, hydration state; all arrangement authority.
- `lib/app-registry.ts` — static registry of the 14 apps (paths, icons, default rects, dock placement, instance matching, preload).
- `lib/os-state-client.ts` — WebSocket client for the `os_shell` domain: snapshot hydration, debounced writes, echo suppression, reconnect, degraded mode.
- `components/` — `DesktopShell`, `MenuBar`, `Dock`, `SessionsRail`, `OsWindow` (chrome: traffic lights + per-window `Topbar` + `TopbarSlotProvider`), `CommandPalette`, `SpacesOverlay`, `WallpaperLayer`, `CompactStack` (compact presentation).
- `apps/<app>/` — one window controller per app replacing the old route glue (location parsing via the domain's shared zod schemas, topbar slots, internal nav callbacks).
- Route leaves under `web/src/routes/` become sync-controllers: reconcile URL → `openOrFocus` and render `null`.

**Data flow:** user gesture → WM store action → (a) render, (b) debounced `OsStateClient.put` → daemon hub → bbolt tx → event fan-out → other clients patch their stores. Navigation: any open/focus/in-window link → `router.navigate()` → route sync-controller → WM store → focused window renders; focus change → `navigate` to the window's stored location. Existing TanStack Query + SSE data layers are untouched — window bodies subscribe to the same caches views use today.

### Architectural Boundaries

- New Go package `internal/clientstate`. Import graph: `internal/daemon` → `clientstate` (wiring); `internal/api` → `clientstate` (handlers); nothing else imports it; it imports no AGH domain packages (only store-agnostic helpers + bbolt). `mage Boundaries` updated in the same commit that adds the package.
- `internal/clientstate` never reaches into SQLite streams; `internal/store` never reaches into bbolt. The boundary rule (ADR-008): **state the daemon must match, filter, join, or transact relationally → SQLite with numbered migrations. The desktop-state *envelope* (workspace, key, rev, commit seq, tombstones, deterministic errors) is a typed public contract implemented by `clientstate`; *payloads* stay opaque JSON the daemon never interprets.** Any field that ever needs server-side matching graduates to SQLite with a migration — no query features are ever added to `clientstate`.
- `clientstate` depends on a `WorkspaceResolver` interface (implemented by the workspace service, wired in `internal/daemon`) — the only inward dependency, and it is an interface owned by `clientstate` (no package cycle).
- Web: `systems/os` may import other systems **only** via their public barrels (`@/systems/<domain>`); no system imports `systems/os` (one-way: os → domains). Route files import `systems/os` controllers. `packages/ui` receives tokens + generic primitives only (no WM logic); `react-rnd` is a `web/` dependency exclusively.
- `web/CLAUDE.md` dependency flow inside `systems/os` stays `adapters → lib → hooks → components`; the WS client sits in `lib/` with an injectable socket factory (mirroring the existing `eventSourceFactory` test pattern).

## Implementation Design

### Core Interfaces

Daemon service (final signatures):

```go
package clientstate

type WorkspaceID string

type WorkspaceGeneration string

// WorkspaceResolver is implemented by the workspace service and wired in daemon/.
// A generation uniquely identifies one registration of a workspace id. Normal
// resolution rejects deleting/unknown ids. Purge resolution succeeds only while
// the matching generation is held behind the workspace deletion gate.
type WorkspaceResolver interface {
    ResolveWorkspace(ctx context.Context, ws WorkspaceID) (WorkspaceGeneration, error)
    ResolveWorkspaceForPurge(ctx context.Context, ws WorkspaceID) (WorkspaceGeneration, error)
}

type Service interface {
    Get(ctx context.Context, ws WorkspaceID, domain, key string) (Entry, error)
    List(ctx context.Context, ws WorkspaceID, domain string) ([]Entry, error)
    // Apply commits all ops in ONE bbolt tx (all-or-nothing), assigns consecutive
    // commit seqs in op order, and publishes them as one ordered batch. A single
    // put/delete is a 1-op Apply.
    Apply(ctx context.Context, ws WorkspaceID, domain string, ops []Op, opts ApplyOptions) ([]Entry, error)
    // Watch is linearized: it registers with the workspace sequencer, snapshots at
    // as-of seq S inside the registration window, and delivers only events with
    // seq > S, in commit order. No gap, no duplicate, regardless of concurrent writes.
    Watch(ctx context.Context, ws WorkspaceID, domains []string) (Subscription, error)
    PurgeWorkspace(ctx context.Context, ws WorkspaceID) error
}

type OpKind uint8
const (
    OpPut OpKind = iota + 1
    OpDelete
)

type Op struct {
    Kind  OpKind
    Key   string
    Value []byte // OpPut only
    IfRev uint64 // 0 = last-writer-wins; >0 = CAS against current Rev (tombstone revs count)
}

type ApplyOptions struct {
    Origin string // connection id for echo suppression; "" for HTTP/UDS/CLI writers
}

type Subscription interface {
    AsOfSeq() uint64      // snapshot fence: events delivered have Seq > AsOfSeq
    Snapshot() []Entry    // live entries at AsOfSeq (tombstones excluded)
    Events() <-chan Event // commit order; closed on Close, ctx cancel, or eviction
    Err() error           // non-nil after close when evicted (ErrSlowConsumer)
    Close() error
}
```

```go
type Entry struct {
    Domain    string
    Key       string
    Value     []byte // opaque JSON payload; nil on tombstones
    Rev       uint64 // monotonic per (workspace, domain, key); survives delete via tombstone
    Seq       uint64 // workspace-scoped commit sequence; total order across keys
    Deleted   bool   // tombstone marker
    UpdatedAt time.Time
}

type Event struct {
    Workspace WorkspaceID
    Entry     Entry // Entry.Deleted=true carries the delete
    Origin    string
}
```

Sentinel errors (wrapped with `%w`, mapped 1:1 to API error codes): `ErrNotFound`, `ErrWorkspaceNotFound`, `ErrRevConflict`, `ErrValueTooLarge`, `ErrKeyQuota`, `ErrInvalidDomain`, `ErrInvalidKey`, `ErrInvalidValue`, `ErrEmptyApply`, `ErrSlowConsumer`, `ErrClosed`.

Web WM store (the contract every shell component and controller depends on):

```ts
export type OsAppId =
  | "dashboard" | "session" | "tasks" | "agents" | "network" | "loops" | "jobs"
  | "triggers" | "marketplace" | "bridges" | "knowledge" | "sandbox" | "vault" | "settings";

export interface OsRect { x: number; y: number; w: number; h: number }
export interface OsWindowLocation { pathname: string; search: Record<string, unknown> }
// ADR-009: fractions of the desktop work area, each 0..1 (bounds + minimum-size validated at the codec)
export interface OsSnapZone { fx: number; fy: number; fw: number; fh: number }

export interface OsWindow {
  id: string;                      // "app:<appId>" | "session:<sessionId>"
  app: OsAppId;
  instanceKey: string | null;      // sessionId for session windows, null otherwise
  location: OsWindowLocation;      // current location inside the app's subtree
  rect: OsRect;                    // floating-mode geometry (last committed px; derived while snapped/maximized)
  prevRect: OsRect | null;         // zoom/snap restore target
  snap: OsSnapZone | null;         // ADR-009: while set, geometry derives locally from the viewport
  z: number;
  minimized: boolean;
  maximized: boolean;
}
```

```ts
export interface OsDesktopStore {
  windows: Record<string, OsWindow>;
  focusedId: string | null;
  railOpen: boolean;
  wallpaper: "ember" | "mesh" | "carbon";
  presentation: "floating" | "compact";        // derived from viewport (960px)
  hydration: "pending" | "live" | "degraded";  // degraded = WS unavailable, in-memory only
  openOrFocus(target: { app: OsAppId; instanceKey?: string; location?: OsWindowLocation }): string;
  closeWindow(id: string): void;
  focusWindow(id: string): void;
  minimizeWindow(id: string): void;
  restoreWindow(id: string): void;
  toggleZoom(id: string): void;
  snapWindow(id: string, zone: OsSnapZone): void; // ADR-009: sets snap + prevRect, clears maximized; commitRect clears snap
  toggleRail(): void;                          // sessions rail; dock sessions item + palette call this
  commitRect(id: string, rect: OsRect): void;   // gesture-end commit; transient moves stay local
  setLocation(id: string, loc: OsWindowLocation): void; // route sync-controller only
  applyRemote(event: OsStateEvent): void;       // WS patches; rev-guarded
}
```

App registry entry:

```ts
export interface OsAppDefinition {
  id: OsAppId;
  title: string;
  icon: LucideIcon;
  paths: string[];                    // route prefixes owned, e.g. ["/loops", "/loop-runs"]
  defaultRect: OsRect;                // hand-tuned cascade from the prototype
  dock: { group: 1 | 2 | 3 | 4 } | null; // null = settings (menubar) & session (rail-opened)
  badge?: "sessions" | "tasks";       // wired to the existing nav-counts store
  matchInstance?: (pathname: string) => string | null; // session: extracts sessionId
  preload?: (qc: QueryClient, ctx: { workspaceId: string }) => Promise<void>; // reuses -*-preload.ts
  Controller: React.ComponentType<{ windowId: string }>;
}
```

### Data Models

**bbolt layout** (`$AGH_HOME/state/clientstate.db`, file mode 0600, opened at daemon boot, closed on shutdown):

- bucket `ws:<workspace_id>` — one per workspace; deleted whole by `PurgeWorkspace`.
  - key `__meta.seq` — the workspace commit sequence (uint64 BE), incremented inside every committing tx.
  - bucket `<domain>` — internal axis; **v1 handlers fix it to `os_shell`** (ADR-008); validated `[a-z0-9_-]{1,64}`.
    - `<key>` → envelope; key validated `[a-zA-Z0-9:._-]{1,128}`. Deletes write a **tombstone** envelope (rev+1, no payload) so revision continuity survives delete→recreate; tombstones are removed only by `PurgeWorkspace`.

**Ordering machinery (peer-review B-002):** all writes for a workspace funnel through one per-workspace sequencer goroutine that (1) commits the tx, (2) publishes the batch to the hub — in that order, serialized, so hub delivery order always equals commit order. `Watch` registers with the same sequencer, snapshots at the current seq, and receives only later commits — making snapshot+stream composition gap-free by construction rather than by client-side fencing.

Envelope encoding (binary header + payload; no per-write JSON re-marshal of metadata):

| Field | Shape | Purpose |
|---|---|---|
| rev | uint64 BE | Monotonic per key; incremented inside the same bbolt tx; tombstones keep the counter alive across delete→recreate |
| seq | uint64 BE | Workspace commit sequence stamped in the same tx; the total order used for fences and LWW settlement |
| flags | uint8 | bit0 = tombstone |
| updated_at | int64 BE (unix nanos) | Daemon clock at commit; surfaces staleness to clients and CLI |
| payload | raw JSON bytes | Opaque to the daemon; size-capped by config; schema owned by the writing client; absent on tombstones |

**Canonical wire DTO (N-002)** — defined once in `internal/api/contract`, generated everywhere: `{key: string, value: object|null, rev: number, seq: number, deleted: bool, updated_at: RFC3339}`. Revisions/seqs are JSON numbers guarded ≤ 2^53−1 (daemon lifetime cannot realistically exceed it; guard asserts anyway). `value` must be a JSON **object** (not scalar/array) for `os_shell`. List responses sort by `key` ascending. Tombstones never appear in list/get/snapshot — only as `deleted: true` events.

**Storage-shape decision (side-table vs JSON):** this state is deliberately an opaque JSON payload in a KV store, not SQLite columns or a side-table, because nothing in it is server-side matchable — the daemon never filters, joins, or queries desktop layout. That inverts the usual AGH rule (side-tables for matchable state, L-003) in the only direction that rule permits: JSON blobs are legitimate precisely when the payload is opaque. The public contract lives in the typed **envelope** (key, rev, seq, tombstones, errors), not in payload interpretation (ADR-008). The moment any field needs SQL matching, it graduates to SQLite with a numbered migration; `clientstate` never grows query features (see Architectural Boundaries).

**`os_shell` domain payloads** (client-owned schemas, versioned with `v`; the daemon never parses them):

- key `desktop` → `{ v: 1, focusedId, railOpen, wallpaper, dockMagnify, reduceMotion }` — one small doc for desktop-wide prefs; changes are infrequent, so a single key avoids fan-out chatter.
- key `win:<windowId>` → `{ v: 1, app, instanceKey, location, rect, prevRect, z, minimized, maximized, snap }` — one key **per window** so concurrent clients editing different windows never conflict, rect churn touches only the moved window's key, and window close maps to a KV delete. `snap` (ADR-009) is `null` or `{fx,fy,fw,fh}` work-area fractions; it always travels inside the whole `win:*` doc, so a rect-only write is by construction a write with `snap: null` — the two can never disagree.
- **Invariant-spanning actions ship as one `apply` batch** (peer-review B-003): focus = `desktop` + focused `win:*` (z bump); close-focused = delete `win:*` + `desktop` (successor focus); zoom = one `win:*`; workspace-switch persistence = batch. Cross-key window invariants are therefore atomic at the store, on the wire, and in the daemon tx.

**Config keys** (`config.toml`, section `[desktop_state]` — ADR-008 naming):

These are daemon-global limits because one store and one bounded write path serve every workspace. Global `config.toml` and global config writes may set them; workspace overlays and workspace-scoped writes are rejected before persistence.

| Field | Default (range) | Purpose |
|---|---|---|
| max_value_kib | 256 (1..4096) | Per-entry payload cap; oversized ops fail `ErrValueTooLarge` |
| max_keys_per_workspace | 512 (16..8192) | Per-workspace retained-key quota, including tombstones; a new key identity beyond the limit fails `ErrKeyQuota`, while updates or recreation of an existing identity always pass |

**WebSocket frames** (JSON; schemas registered as OpenAPI components so generated TS types cover them; every client mutation carries a client-generated `req` id echoed on ack/error — peer-review B-003):

- client→server: `{"op":"sub"}` · `{"op":"apply","req":string,"ops":[{"kind":"put"|"delete","key","value"?,"if_rev"?}...]}` (single put/delete = 1-op apply) · `{"op":"ping"}`
- server→client: `{"op":"snapshot","as_of_seq":number,"entries":[DTO...]}` (once, after `sub`) · `{"op":"event","entry":DTO,"origin":string}` (commit order, `seq > as_of_seq`) · `{"op":"ack","req":string,"results":[{"key","rev","seq"}...]}` · `{"op":"error","req"?:string,"code":string,"key"?:string}` · `{"op":"pong"}`

**Connection lifecycle (peer-review B-004):** one reader pump + one single-writer pump per connection (gorilla/websocket requires serialized writes); bounded outbound queue (256 frames); write deadline 10s; ping/pong idle policy 30s/60s. Eviction sends a best-effort `error{code:"desktop_state_slow_consumer"}` with a 2s deadline, then closes — a stalled socket can never block the hub, other subscribers, or writers. Frame-triggered mutations execute under a **daemon-owned lifecycle context with a 5s per-op deadline** (derived from the daemon base context, detached from the request context) — disconnect never aborts a committing tx, and daemon shutdown still cancels queued work. Shutdown order: stop accepting upgrades → close subscriptions → join pumps (WaitGroup) → close store.

### API Endpoints

Public surface is **desktop-state** (ADR-008 — no domain on the wire; handlers fix the internal domain to `os_shell`). HTTP and UDS serve identical routes **including the stream upgrade** — the WS endpoint is registered on both listeners, so watch parity is real, not CRUD-only (peer-review B-008). All added to OpenAPI + generated TS + generated CLI docs in the same change (`eng-contract-codegen-coship`):

- `GET    /api/workspaces/{workspace_id}/desktop-state` → `200 {as_of_seq, entries: DTO[]}`.
- `GET    /api/workspaces/{workspace_id}/desktop-state/{key}` → `200 DTO` | `404 code=desktop_state_not_found`.
- `PUT    /api/workspaces/{workspace_id}/desktop-state/{key}` body `{value: object, if_rev?: number}` → `200 DTO` | `409 code=desktop_state_rev_conflict` | `413 code=desktop_state_value_too_large` | `422 code=desktop_state_key_quota_exceeded | desktop_state_invalid_key | desktop_state_invalid_value`.
- `POST   /api/workspaces/{workspace_id}/desktop-state/apply` body `{ops:[...]}` → `200 {results}` (atomic batch; same failure shapes).
- `DELETE /api/workspaces/{workspace_id}/desktop-state/{key}` (`?if_rev=`) → `204` | `404` | `409`.
- `GET    /api/workspaces/{workspace_id}/desktop-state/stream` → WebSocket upgrade (frames above), served on HTTP **and** UDS listeners.
- Unknown/deleted workspace on any route → `404 code=workspace_not_found` (peer-review B-005).

CLI (structured output via the repo-standard `-o` flag — peer-review N-001; identity-explicit operator verbs):

- `agh desktop-state list --workspace <id> [-o json]`
- `agh desktop-state get --workspace <id> --key <k> [-o json]`
- `agh desktop-state set --workspace <id> --key <k> (--value <json> | --file <path>) [--if-rev N] [-o json]`
- `agh desktop-state delete --workspace <id> --key <k> [--if-rev N]`
- `agh desktop-state watch --workspace <id> [-o jsonl]` → ordered event stream over the UDS upgrade endpoint (same snapshot fence semantics as the browser).

## Routing Model (URL ↔ Window Manager)

One **routing coordinator** (`systems/os/lib/routing-coordinator.ts`) mediates every URL↔WM transition with an explicit cause: `user-open` · `user-focus` · `route-pop` · `hydrate` · `workspace-switch` (peer-review B-006, ADR-002 amendment). The WM store is **side-effect-free** — no store action ever navigates.

1. **Route leaves become sync-controllers.** Each route file keeps its path, `validateSearch`, `beforeLoad` crumb context, and `loader` (preload). Its component becomes `createOsRouteSync(app)` — it reports the matched location to the coordinator as `route-pop` (or the initial-URL intent during `hydrate`) and renders `null`. The desktop (`win-layer`) renders windows outside the `Outlet`.
2. **Route-originated reconciliation never writes history.** On `route-pop`/deep link, the coordinator updates the store (open-or-focus + location) only — it never calls `navigate`. This breaks the feedback cycle: browser Back can never re-push what it just popped.
3. **User-originated transitions write exactly one history entry.** Dock click, palette selection, rail row, `<Link>` clicks, and `user-focus` each produce a single `router.navigate({replace:false})` issued by the coordinator; the resulting route match reconciles the store without a second write. A pointer-focus immediately followed by an in-window link yields the link's entry only (focus during pointerdown coalesces with the click navigation in the same tick).
4. **Hydration precedes URL intent.** On boot the coordinator applies the daemon snapshot first (`hydrate`), then applies the initial URL as the final focus intent — a cold deep link always wins over the restored focus, and never loses the rest of the restored desktop.
5. **Activation focuses, pointer or keyboard.** Any interaction that moves DOM focus into an unfocused window (pointerdown, Tab, Enter on a focusable) triggers `user-focus` — keyboard users get the same in-window link semantics as pointer users.
6. **Unfocused windows never touch route hooks.** Controllers read `location` from the WM store and parse `search` with the same shared zod schemas the route uses; `useParams`/`useSearch` are confined to sync-controllers. On mount (hydration restore), controllers call the app's `preload` directly to warm the query cache — the same functions loaders call.
7. **The session redirect route** `/session/$id` keeps resolving agent → `/agents/$name/sessions/$id`, which the agents sync-controller maps to `session:<id>` via `matchInstance`.
8. **Workspace switch** (menubar/palette/cross-workspace deep link) runs as `workspace-switch`: persist the outgoing space (one `apply` batch), swap the store to the target space, then one navigate to that space's focused location (or `/`). The `network` app's `$workspaceId` path segment always uses the active workspace.

## Presentation Modes

- `floating` (≥960px): windows render inside `react-rnd` (controlled), clamped to the desktop layer with menubar/dock gutters; re-clamped on viewport resize (fixes a prototype gap). Dock magnification, wallpaper, and genie-minimize (motion) are floating-mode behaviors.
- `compact` (<960px): the same logical windows render as a full-viewport stack (`CompactStack`) — focused window fills the viewport; rect/drag/resize/zoom are disabled (state preserved for return to floating). **The visual reference exists in the prototype's compact block** (`os-v2.css:557-628` + `os-v2.js` `mqMobile`): dock becomes a 56px bottom tab bar (horizontal scroll, no magnification/tooltips/separators, safe-area insets), zoom control hidden, traffic lights enlarge hit areas (15px + inset expansion), menubar menus collapse, palette/bell/toasts/spaces clamp to the viewport. Compact implementation is a normal Visual Contract Mode phase against this reference. The sessions rail presents as a full-height overlay sheet in compact (delta recorded below — the prototype keeps the fixed rail).
- Mode is derived (media query), never persisted; switching modes loses no logical state.
- Minimize semantics (both modes): the window **body unmounts** (streams and subscriptions in it tear down); the WM entry survives with `minimized: true` + rect. Restore remounts; TanStack Query cache makes content return instantly. This is the N-windows performance posture: cost scales with *visible* windows.

## Window Snap Layer (ADR-009)

Floating-mode arrangement accelerator — FancyZones-class snap on free-floating windows. The Hermes tiling tree is explicitly NOT ported (see `analysis/02_analysis_hermes-window-arrangement.md` + ADR-009 follow-ups); what transfers is its zone-resolver pattern and its motion constants.

- **Zones (v1):** left/right halves (`{0,0,.5,1}` / `{.5,0,.5,1}`) and four corner quarters. The top edge stays unbound — zoom already owns "fill the desktop"; a top-edge zone would create a second maximize path.
- **Gesture:** dragging a window within the sensitivity radius (20px) of a desktop edge/corner publishes exactly one zone hint per frame (pure resolver against rects snapshotted at drag start); the drop overlay fades in at 80ms linear and morphs between targets at 150ms ease-out transitioning only insets/background/border/opacity (never `transition-all`); backdrop blur applies to the active target only; Escape or releasing outside every zone cancels with geometry unchanged; release commits `snapWindow`. Constants are adopted verbatim from the Hermes FancyZones port and are the **normative motion contract** — Storybook + `eng-ui-screenshot` captures + designer review are the parity evidence; no rendered-reference bundle (the Hermes paradigm differs; its constants, not its pixels, are the contract).
- **Derived geometry:** while `snap` is set, the rendered rect = desktop work area (desktop layer minus menubar/dock gutters) × fractions, computed locally per client on the same non-Rnd absolute path maximized windows use, clamped to window minimums (280×180 — fraction overflow accepted on short viewports). Viewport resize re-derives and **never commits** (no cross-client oscillation between viewport sizes); proportional reflow is therefore free. `rect` keeps the last committed px (the snapping client's derivation at commit time) for thumbnails and non-rendering readers.
- **Exclusivity + restore (invariant 19):** `snap` and `maximized` are mutually exclusive — setting either clears the other. `snapWindow` stores the pre-snap rect in `prevRect`; dragging a snapped window away (or any `commitRect`) clears `snap` and restores pre-snap dimensions under the pointer; keyboard/palette restore returns `prevRect` exactly.
- **Keyboard + palette:** palette commands for every zone + restore ("Snap left half", …); chords ⌃⌥←/→ (halves), ⌃⌥U/I/J/K (quarters), ⌃⌥↓ (restore); Help menu lists them; the palette is the universal fallback where a browser reserves a chord and the guaranteed keyboard path for a11y. Compact presentation: snap actions are no-ops and no snap affordance renders (UT-061 gating pattern).
- **Agents:** an agent snaps a window by writing the `win:*` doc with `snap` fractions via `agh desktop-state set` / HTTP / UDS — any fraction rect within bounds/minimums is valid, a strictly larger surface than the six pointer zones. Codec validation salvages invalid `snap` to `null` without dropping the window.

## Attention Surfaces (truthful contracts — peer-review B-007)

Every attention affordance names its runtime projection; nothing renders without one (SD-007):

- **Dock badge · sessions** = count of sessions in a waiting state, computed from the canonical **session catalog cache** (already streamed via `session_catalog_changed` SSE + `sessionKeys` invalidation). Client-side counting over runtime-owned data; stale when the catalog stream reports stale.
- **Dock badge · tasks** = `awaiting_approval_tasks` from the tasks dashboard snapshot (`internal/api/contract/tasks.go:430`, already fetched by the nav-counts reconciliation path). NOT the current nav-counts `tasks` figure (task-activity total) — the dock badge means "needs you", so it binds to the awaiting-approval projection. The nav-counts store keeps its 8 keys for other consumers; the dock badge adapter reads the two named projections above.
- **Menubar bell** = an **aggregator, not an approval authority**: it lists waiting sessions (catalog) and awaiting-approval tasks (dashboard), each row a focus action that opens/focuses the owning window where the runtime-backed approve/deny surfaces already live. No inline approve/deny in v1 (prototype delta recorded below); no new backend endpoint is required.
- **Sessions rail** = store actions `toggleRail()/openRail()/closeRail()` own `railOpen`; the dock `sessions` item and the palette both trigger them (registry marks the sessions app `dock: rail-toggle`). Rail status vocabulary maps 1:1 to session catalog statuses. Compact: rail renders as an overlay sheet.
- Badge display truth: zero renders nothing; counts cap at "9+" without collapsing the zero/non-zero distinction; a stale source hides the badge rather than showing a stale number.

## Modal & Overlay Policy (owner decision, 2026-07-19)

Today every `Dialog`/`Sheet`/`ConfirmDialog` portals to `document.body` with a full-viewport scrim — correct in a single-page shell, broken on a desktop (a confirm in the tasks window would freeze the session windows beside it). The policy:

1. **Window-scoped by default.** A modal surface opened from window content belongs to that window: it portals into the **owning window's container**, renders centered and clamped to the window bounds (min gutter 16px), its scrim covers only that window, and it travels with drag. Other windows stay fully interactive; focusing another window leaves the dialog waiting in its own window (macOS-sheet semantics, centered visual).
2. **The seam is one context, not a migration.** `OsWindow` provides an `OverlayContainerContext`; the `@agh/ui` `Dialog`/`Sheet`/`AlertDialog` portal wrappers read it with a `document.body` fallback. The ~40 existing dialog call sites across `systems/*` change **zero lines** — mounted inside a window they scope automatically; outside (design-system route, onboarding) they behave as today. Focus containment is scoped to the dialog while its window is focused — no global `inert`, no global focus lock. Stacking is inherited: each window is its own stacking context, so window A's dialog can never paint above focused window B.
3. **Route-backed "modals" become in-window locations.** `/tasks/new` and `/tasks/$id/edit` render as internal locations of the tasks window (window-local breadcrumb deepens: `Tasks / New`), deep-linkable by construction; the network thread overlay stays a route-driven overlay inside the network window. Rule of thumb: wizard-class flows (`--width-modal-lg/xl` tokens exceed common window widths) prefer internal navigation; short forms and confirms stay window-scoped dialogs clamped to the window.
4. **The desktop-level overlay set is closed:** command palette, Spaces overview, toasts, the onboarding/workspace gate, and confirms that destroy their own context (e.g. workspace delete). Everything else is window-scoped; adding to this list is a spec change.
5. **Minimize protection.** A window with an open modal dialog is exempt from minimize-unmount: minimize hides it (dock indicator as usual) but keeps it mounted until the dialog closes or submits — unsaved form state survives. Restore returns with the dialog exactly as left.
6. **Compact:** the window fills the viewport, so window-scoped modals are naturally full-viewport; no extra design.

## Design Tokens & Visual Contract

- New tokens land in `packages/ui/src/tokens.css` (canonical source): `--shell-glass`, `--shell-glass-pop`, `--radius-window: 12px`, `--radius-dock: 22px`, `--shadow-window`, `--shadow-window-unfocused`, dock item metrics. `make codegen` regenerates `DESIGN.md`; the flat-depth rule gains an explicit, documented **OS-shell chrome exception** (glass + window shadows apply to shell chrome — menubar, dock, rail, popovers, window frames — never to content inside window bodies). Never hand-edit `DESIGN.md` generated regions.
- Window head: the route `PageHead` is absorbed into the OS chrome per `docs/design/opendesign/os/pagehead-redesign.html` and `OS-V2-DELIVERY.md` — one 44px unified bar (traffic lights · quiet 22px glyph + 13px/600 title · drag flex · status chip + ≤2 actions) with an optional 38px context strip for listing tools. Identity renders once; workspace breadcrumbs are deleted (menubar owns workspace). Drill-in swaps the glyph for back + window-local crumbs; document windows (sessions) self-title with a state dot. Reuse `@agh/ui` `Topbar` + per-window `TopbarSlotProvider`, evolved to this anatomy — not a second body hero.

**Visual-contract delta table (peer-review N-007/B-009)** — every intentional deviation from the prototype reference, exhaustively:

| # | Prototype behavior | Product behavior | Why |
|---|---|---|---|
| 1 | Window head `min-height: 44px` in CSS | 44px unified head (PageHead absorbed; `pagehead-redesign.html`) | Hard-cut: delivery + redesign supersede the earlier 48px Topbar delta |
| 2 | Boot fallback auto-opens the first session | First run = empty desktop + ⌘K/dock hint | PRD US-001.EC-1: nothing opens uninvited |
| 3 | Bell popover has inline Approve/Deny | Bell rows focus the owning window | B-007: v1 bell is an aggregator; approve/deny stays on runtime-backed surfaces |
| 4 | Dock badges seeded on sessions+tasks demo data | Badges bind to named projections (waiting sessions; `awaiting_approval_tasks`) | SD-007 truthful counts |
| 5 | Compact keeps the fixed sessions rail | Rail becomes an overlay sheet in compact | 296px fixed rail does not fit <960px stacked windows |
| 6 | Scripted demo event flips a session at 14s | None | Demo-only |
| 7 | No window re-clamp on browser resize | Windows re-clamp on viewport resize | Prototype gap fixed (US-002.EC-2) |
| 8 | zTop/focus/maximized not persisted | Full persistence incl. focus + maximized via desktop-state | Continuity guarantees |
| 9 | Single session window (title mutates) | Multi-instance `session:<id>` windows | ADR-002; core product value |
| 10 | localStorage persistence | Daemon desktop-state (ADR-004/008) | Owner decision |
| 11 | No dialog/modal handling in the prototype | All dialogs/sheets window-scoped via `OverlayContainerContext` (Modal & Overlay Policy) | Multi-window interactivity is the core value; desktop-blocking modals would negate it |
| 12 | Menubar mark is a bespoke `mb-logo` tile (accent square + ink dot) | Real `@agh/ui` `Logo` `symbol` at `--size-menubar-logo` | Prototype marks are placeholder art; the brand inventory owns marks — placeholders never become `Logo` variants (L-032) |
| 13 | Active space is identified only by an accent border | Active cards also render a visible `Current` pill | Status must never depend on color alone |
| 14 | Compact traffic lights use overlapping 33px pseudo-hitboxes | Close/minimize use non-overlapping 44px targets around the 15px glyphs | Comfortable touch targets and deterministic hit ownership take precedence over literal spacing parity |
- Traffic lights: 12×12, 4px radius, 7px gap; colors on hover/focus only (accent/muted/success per prototype CSS:416-424).
- Signal palette unchanged; wallpapers are tokenized gradient stacks (ember/mesh/carbon).
- Every shell surface ships a Storybook story; `eng-ui-screenshot` captures are completion evidence; the floating shell is verified against the prototype as a rendered reference/implementation bundle (Visual Contract Mode).

## Safety Invariants

Sync + window-manager invariants, numbered (enforced in code and asserted by the `_tests.md` cases that cite them):

1. `Rev` increments by exactly 1 per committed write of its `(workspace, domain, key)` — including tombstone writes — inside the same bbolt tx; rev continuity survives delete→recreate (tombstones removed only by purge). CAS against a tombstone's rev is valid.
2. `Seq` is a workspace-scoped total order stamped in the same tx (`__meta.seq`); the per-workspace sequencer serializes commit→publish so hub delivery order equals commit order, always.
3. `Watch` is linearized: registration and snapshot happen inside the sequencer window; the subscription's `AsOfSeq` fences it — delivered events all have `seq > AsOfSeq`, in seq order, gap-free and duplicate-free under any concurrent write/delete/create.
4. `Apply` is atomic: all ops commit in one tx with consecutive seqs or none do; partial application is impossible at every layer (store, wire, client).
5. Slow consumers never block writers: bounded per-subscription buffer (256); overflow evicts (`ErrSlowConsumer` → best-effort WS `error` frame under a 2s write deadline, then close); the client re-syncs via fresh `sub` + snapshot.
6. Echo suppression is by `Origin` connection id only; the writing client tracks pending mutations by `req` — remote events touching a pending key are buffered and settled by `seq` order once the ack arrives (LWW by seq, no flicker loops).
7. `if_rev` CAS failures return `ErrRevConflict` and change nothing; last-writer-wins (`if_rev` absent) is the default for `os_shell` geometry keys.
8. Every socket has exactly one writer pump; frame-triggered mutations run under the daemon lifecycle context with a 5s per-op deadline — disconnect never aborts a committing tx (L-001), and daemon shutdown cancels queued work and joins both pumps before the store closes (SD-010 in both directions).
9. Every operation resolves the workspace at admission and revalidates the returned `WorkspaceGeneration` inside its workspace-sequencer turn: unknown/deleting ids or a changed generation fail `ErrWorkspaceNotFound`; nothing is created implicitly. Workspace deletion runs a serialized gate — mark-deleting (normal resolution rejects, purge resolution admits only the fenced generation) → close subscriptions → purge → row delete — so a delayed operation from an old registration can never resurrect state in a recreated same-id workspace.
10. The daemon is authoritative on reconnect: the snapshot replaces the local mirror for all keys **except** keys locally modified while degraded, which the client re-applies as one `apply` batch after adopting the snapshot (LWW by seq — "what I see is what persists"). This is the degraded-recovery policy, verbatim in PRD US-013.
11. Payloads are opaque: the daemon validates size, identifiers, and JSON-object shape only; `os_shell` schema evolution is client-owned via the `v` field and requires no daemon change.
12. WM z-order: the focused window always holds max `z`; `z` values compact to `1..N` on hydration so they never grow unbounded.
13. The routing coordinator is the only URL↔WM bridge (Routing Model rules 1–5): store actions never navigate; route-originated causes never write history; user-originated causes write exactly one entry; hydration precedes URL intent (L-005 applied to the frontend).
14. Single-instance guard lives in the store: `openOrFocus` for an existing `(app, instanceKey)` focuses and returns the existing id — call sites cannot duplicate windows.
15. Rect persistence is debounced (250ms trailing) with a mandatory flush on gesture end (`onDragStop`/`onResizeStop`) and on `beforeunload`; transient gesture positions are never written; invariant-spanning actions persist as `apply` batches (never split writes).
16. In `degraded` hydration (WS unavailable), the shell runs fully from in-memory state, surfaces a non-blocking indicator, and retries with backoff; no feature is gated on persistence availability.
17. Attention surfaces render only named runtime projections (Attention Surfaces section); a stale source hides its badge instead of showing a stale count.
18. Modal surfaces are window-scoped by default (Modal & Overlay Policy): they portal into the owning window's container, block only that window, and inherit its stacking context; the desktop-level overlay set is closed. A window with an open modal dialog is exempt from minimize-unmount until the dialog closes.
19. Window geometry has one derived authority per state (ADR-009): `snap` and `maximized` are mutually exclusive — setting either clears the other; while either is set, rendered geometry derives from the local viewport and viewport resize never commits; every snap mutation ships as a whole `win:*` doc write so `snap` and `rect` can never disagree; codec validation salvages invalid `snap` (out-of-range fractions, sub-minimum zones) to `null` without dropping the window.

## Impact Analysis

| Component | Impact Type | Description and Risk | Required Action |
|-----------|-------------|---------------------|-----------------|
| `internal/clientstate` | new | KV service, bbolt store, hub; low risk, isolated | Build + unit/integration tests + boundaries entry |
| `internal/api` (contract, spec, core) | modified | New CRUD routes, WS endpoint, error codes; medium risk (first browser WS) | Handlers + OpenAPI + codegen + tests |
| `internal/cli` | modified | New `desktop-state` verb family; low risk | Verbs + `-o` output + tests + generated docs |
| `internal/daemon` | modified | Wiring (service + resolver + both-listener stream) + workspace-delete gate/purge; low risk | Composition + boot/shutdown lifecycle |
| `config.toml` surface | modified | `[desktop_state]` keys; low risk | Structs/defaults/validation/docs/example/tests |
| `web/src/systems/os` | new | Entire shell + WM + sync client; high complexity | Full build per Development Sequencing |
| `web/src/routes/*` | modified | Leaves become sync-controllers; medium risk (URL semantics) | Per-app port tasks with e2e coverage |
| Route glue (`-*-route.tsx`, `use-*-route.ts`) | deprecated | Replaced by window controllers; deleted per app | Delete in each port task |
| `systems/runtime` (sidebar, ⌘K registry, nav-counts) | modified | Sidebar deleted; nav-counts rewired to dock badges; ⌘K freed | Delete targets sweep + dock badge adapter |
| `systems/runtime` RuntimeSelector hosts | modified | RuntimeSelector rebinds ⌘J within the nearest real form/dialog host; low risk | Rebind + Kbd hint update; do not invent per-prompt runtime controls |
| `packages/ui` | modified | New shell tokens (+ codegen `DESIGN.md`); `Dialog`/`Sheet`/`AlertDialog` portal wrappers read `OverlayContainerContext` (body fallback — zero call-site changes); low risk | tokens.css + `make codegen` + portal-context wiring + stories |
| `packages/site` docs | modified | Desktop-state API/CLI reference + OS shell concept page | Docs tasks (generated refs + authored page) |
| `skills/agh` (official skill) | modified | Document `agh desktop-state` + endpoints | Skill update in contract task |
| `docs/qa/scenarios` | modified | Shell supersedes chrome scenarios; new surface | New `untested` scenarios; reset superseded ones |
| `DESIGN.md` authored prose (§1, §4, §5, §6, §10) | modified | §4 describes the deleted rail+sidebar+content frame; §1/§5 forbid blur/shadows the shell chrome now uses; medium risk of guardrail conflict | Rewrite layout grammar to desktop model; document the two-layer depth carve-out (shell chrome glass/shadows vs flat content) and shell motion tier — same change as the tokens |
| `COPY.md` §6 + `docs/_memory/glossary.md` | modified | New public vocabulary (`window`, `desktop`, `dock`, `menubar`, `space`, `desktop state`) + new CLI verb family trigger §12 update rules; `desktop state` must stay unconfusable with `memory` (N-006) | Add vocabulary rows + glossary entries in the copy/docs task |
| `PRODUCT.md` §Anti-references | modified | "decorative glassmorphism" anti-reference needs the shell-chrome nuance (tokenized shell glass is sanctioned; content glass stays banned) | One-line clarification |
| `web/CLAUDE.md` design rules | modified | "Flat depth only … no glass except the sticky site header" contradicts the shell chrome | Update the rule to reference the DESIGN.md shell carve-out in the tokens change |

## Extensibility Integration Plan

- **New surface, deliberately narrowed (ADR-008):** the public wire exposes only `desktop-state` — no domain parameter, no reserved public namespace, no extension API in v1. The internal `clientstate` domain axis exists solely so a future consumer ships as its own TechSpec with its own surface. This resolves the "prematurely generic" concern: nothing public dangles unused.
- **Unchanged and verified:** extension manifests, hooks catalog, skills/capabilities surfaces, tools/resources, bundles, registries, bridge SDKs, and MCP sidecars — the shell is a presentation refactor; no runtime extension point changes. Checked surfaces: hook event catalog, extension Host API method list, native toolsets, bundle/registry schemas.
- **Protocol docs:** desktop-state HTTP/UDS/WS contracts land in the generated API reference; WS frame schemas ship as OpenAPI components.

## Agent Manageability Plan

- **CLI:** `agh desktop-state list|get|set|delete|watch` (`-o json`, watch `-o jsonl`) — agents can inspect and arrange any workspace desktop (e.g., write `win:*` keys) through the same authoritative surface the web uses.
- **HTTP/UDS parity:** identical routes on both transports **including the stream upgrade** — `watch` semantics (fence, ordering, eviction, errors) are transport-equal and integration-tested (peer-review B-008).
- **Deterministic errors:** `desktop_state_not_found`, `workspace_not_found`, `desktop_state_rev_conflict`, `desktop_state_value_too_large`, `desktop_state_key_quota_exceeded`, `desktop_state_invalid_key`, `desktop_state_invalid_value`, `desktop_state_slow_consumer` — stable codes across HTTP/UDS/CLI/WS.
- **Discovery:** `list` verbs + OpenAPI + generated CLI docs; `skills/agh` updated with the verb family and the `os_shell` payload key conventions.
- **Snap by fractions (ADR-009):** the `win:*` payload's `snap` field is agent-writable — any work-area fraction rect within bounds/minimums, a strictly larger surface than the six pointer zones; documented with the payload conventions in `skills/agh` and the site reference.
- **No native tool in v1:** `agh__*` toolsets unchanged (checked: toolset descriptors, schema digests, capability gates); a desktop-arrangement native tool is a named post-MVP candidate once agent demand exists.

## Config Lifecycle

- **Added:** daemon-global `[desktop_state] max_value_kib = 256` (range 1..4096), `max_keys_per_workspace = 512` (range 16..8192). Same change ships: config structs, defaults, global overlay behavior, workspace-overlay rejection, validation (range + type errors at load), `config.toml` example, generated CLI/site config reference, and unit tests for default/override/invalid values.
- **Removed/renamed:** none.
- **Checked and unaffected:** all existing config sections; no web-side config keys (breakpoint and debounce are code constants — presentation policy, not operator policy).

## Web/Docs Impact

- **web/**: this program *is* the web impact — shell replacement, 14 app ports, palette, WS client. All enumerated above.
- **packages/site**: generated API + CLI references pick up desktop-state; one authored concept/docs page for the OS shell (windows, spaces, shortcuts) and the `os_shell` payload key conventions.
- **QA tracker flag:** user-visible behavior changes massively → add content-addressed `untested` scenarios for: desktop shell chrome, window lifecycle, window snap, multi-session windows, palette, spaces/workspace switching, persistence across reload/multi-tab, compact mode, `agh desktop-state` CLI. Reset to `untested`: existing scenarios asserting the old chrome (e.g. `ET-web-route-chrome-topbar`, tasks-mode-URL chrome assumptions). Flag, don't retest — next QA cycle owns execution.

## AGH Impact Audit

- **Native tools:** no impact — no `agh__*` tool IDs, toolsets, descriptors, schema digests, or capability gates change; agents operate the new surface via CLI/HTTP/UDS (checked: native toolset registry, descriptor digests).
- **Extensibility and hooks:** no hook/extension/skill/tool/resource/bundle/registry/bridge/MCP contract changes; the public wire has no generic namespace (ADR-008) — a future extension-facing state API ships as its own TechSpec (checked surfaces listed in Extensibility Integration Plan).
- **Workspace data isolation:** desktop state is **workspace-scoped**: bbolt buckets keyed by `workspace_id`, every operation gated by the `WorkspaceResolver` (`workspace_not_found` on unknown/deleted ids), WS subscriptions bound to one workspace, serialized deletion gate + purge, and isolation tests prove no cross-workspace read/watch leakage AND no deleted-workspace resurrection. No session/agent-scoped data added; SSE/event/cache paths untouched.
- **Official AGH skill:** `skills/agh/` updated — `agh desktop-state` verb family, HTTP/UDS routes (incl. stream), error codes, `os_shell` payload key convention, palette/shortcut changes (⌘K/⌘J).

## Testing Approach

Strategy only — every concrete case lives in `_tests.md` (IDs referenced by tasks):

- **Go unit** (`go test -race ./internal/clientstate/...` + api/cli packages): service CRUD/rev/CAS/quota/validation/purge against a temp bbolt file; hub fan-out, ordering, echo, slow-consumer eviction; config validation; handler status+body assertions; CLI output shape. Fakes only at I/O boundaries — bbolt runs real (temp dir), WS handlers tested over `httptest` + real `gorilla/websocket` client.
- **Go integration** (`+integration`): full daemon-wired flow — HTTP PUT visible over WS watch, WS put visible over HTTP GET, two-workspace isolation, workspace-delete purge, restart durability (reopen db, revs preserved).
- **Web unit** (Vitest via `bunx turbo run test --filter=./web`): desktop-store actions + invariants 8/10/12/13, the snap layer (zone resolver, derived geometry, exclusivity — invariant 19), app registry instance matching, `OsStateClient` against an injected fake socket (snapshot fence, echo, reconnect, degraded), palette command sources, window controllers' location parsing.
- **Component/Storybook**: chrome primitives (window head, traffic lights, dock, menubar) stories + a11y addon; `eng-ui-screenshot` visual evidence; floating shell verified as a reference/implementation bundle against the prototype (Visual Contract Mode); compact mode likewise once its prototype lands.
- **E2E web** (Playwright, `make test-e2e-web`): the journeys in `_tests.md` — open/drag/resize/persist across reload, snap zones (drag, keyboard, agent-written fractions), multi-session windows, URL deep links + back/forward focus history, palette flows, workspace-space switching, multi-tab convergence, compact viewport, degraded (daemon WS down) posture.
- **Gate:** single full `make verify` at completion (includes `codegen-check` for OpenAPI/TS/DESIGN.md drift and `mage Boundaries` for the new package).

## Development Sequencing

### Build Order

1. `internal/clientstate` (store + sequencer + service + hub + resolver + config) — no dependencies.
2. API contract + handlers + WS endpoint on both listeners + CLI verbs + OpenAPI/TS codegen + `skills/agh` + site reference — depends on 1.
3. Shell tokens in `packages/ui/src/tokens.css` + `make codegen` (`DESIGN.md`) + chrome primitives + stories — parallel with 1-2. Includes the DESIGN.md/web-CLAUDE.md prose amendments (layout grammar, depth carve-out).
4. `systems/os` core: desktop store + app registry + routing coordinator + `OsStateClient` with pending-mutation state machine (fake-socket tested) — depends on 2-3.
5. `DesktopShell` mounted in `_app` + route sync-controllers + palette with ⌘K takeover / RuntimeSelector ⌘J — depends on 4; first runnable shell (dashboard + settings windows) including react-rnd StrictMode spike assertion.
6. Attention surfaces: dock badge adapter (catalog waiting count + `awaiting_approval_tasks`), bell aggregator, sessions rail with `toggleRail` — depends on 5.
7. Listing-app ports (tasks, agents, loops+loop-runs, jobs, triggers, marketplace, bridges, knowledge, sandbox, vault), each deleting its Delete-Targets rows — depends on 5; parallelizable per app.
8. Sessions: multi-instance session windows, reusing the existing composer unchanged — depends on 5-6.
9. Network app port (deepest router coupling) — depends on 5.
10. Window snap layer (ADR-009): store field + derived geometry + zone resolver + drop overlay + palette/keyboard + codec — depends on 5 and on the unified window head (drag-chrome stability).
11. Spaces (per-workspace arrangements + ⇧⌘S overview) + wallpapers/Appearance + dock magnification — depends on 5.
12. Compact presentation — Visual Contract Mode against the prototype's existing compact block — depends on 5-6.
13. Delete-targets sweep (full inventory) + a11y pass (shortcuts, focus, reduced-motion) + perf-envelope scenario + full `make verify`.
14. QA pair (`qa-report` + `qa-execution`) — appended at task generation per `cy-tasks-tail-qa-pair`.

### Technical Dependencies

- `react-rnd` added to `web/` via `bun add` (only new dependency; `motion`, `cmdk`, `gorilla/websocket`, `zustand` already present). `go get go.etcd.io/bbolt` for the daemon.
- Pedro's compact-mode prototype (blocks step 10 only).

## Monitoring and Observability

- Daemon `slog` events (structured fields `component=clientstate`, `workspace`, `key`, `rev`, `seq`, `conn_id`): store open/close, apply (debug), watch subscribe/unsubscribe, slow-consumer eviction (warn), purge + deletion-gate transitions (info), WS connect/disconnect with close reason. Payload contents never log.
- Bounded-cardinality metrics (peer-review N-003), integrated with the existing observability spine: active WS connections, active subscriptions, outbound queue depth (max/percentile), slow-consumer evictions, snapshot size/duration, apply latency, rev-conflict count, reconnect rate, purge failures.
- API layer logs reuse the existing request logging; WS upgrade failures log with deterministic codes.
- Web: `OsStateClient` exposes hydration/connection state on the store (`hydration`), surfaced as the shell's non-blocking degraded indicator; no console logging in production paths.

## Technical Considerations

### Key Decisions

Recorded as ADRs (below); headline trade-offs: single responsive shell over dual shells (maintenance vs compact design scope); app-subtree windows over per-route windows (coherent URL model vs maximal multitasking); react-rnd over custom geometry (proven edge cases vs one single-maintainer dep); bbolt+WS over SQLite/SSE and over embedded NATS (lifecycle fit + real-time without generic infrastructure); ⌘K to the palette (canonical binding vs relearning).

### Known Risks

- **react-rnd under React 19 StrictMode** — daedalOS precedent says fine; step-5 spike asserts it in our tree; fallback (custom geometry / use-gesture) documented in ADR-003.
- **First browser-facing WebSocket** — reconnect/backpressure/proxy behavior is new territory for this app; mitigated by the pump/deadline lifecycle (invariant 8), degraded mode (invariant 16), and real-socket integration tests; the Vite dev proxy must forward WS upgrades (`ws: true`).
- **Sequencer as a throughput chokepoint** — all workspace writes serialize through one goroutine; acceptable for presentation-state volumes (debounced writes), asserted by the apply-latency metric; never generalize this service to high-throughput domains (Architectural Boundaries).
- **Tombstone growth** — one tombstone per deleted key until purge; tombstones consume `max_keys_per_workspace`, so retained key identities are strictly bounded; recreating an existing identity remains allowed and purge clears all.
- **⌘W/⌘M browser reservation** — some browsers won't yield ⌘W; palette lists every window action as the discoverable fallback; documented limitation.
- **Topbar-slot refactor blast radius** — every view publishing slots is touched; mitigated by porting app-by-app with per-app e2e.
- **Network port complexity** — sequenced as its own late task (step 9) so it never blocks the shell.
- **Two storage engines drift risk** — held by the ADR-008 boundary rule (no query features in clientstate, SQL-matchable state graduates to SQLite).

## Architecture Decision Records

- [ADR-001: Single responsive OS shell replaces the AppShell grid (hard cut)](adrs/adr-001.md)
- [ADR-002: Window = app subtree with internal navigation; URL reflects the focused window](adrs/adr-002.md)
- [ADR-003: WM state in Zustand; react-rnd controlled drag/resize; motion for animation](adrs/adr-003.md)
- [ADR-004: Daemon-side desktop-state KV on bbolt with WebSocket sync](adrs/adr-004.md) — amended round 1 (seq/tombstones/pumps/resolver)
- [ADR-005: Global palette owns ⌘K; RuntimeSelector rebinds to ⌘J](adrs/adr-005.md)
- [ADR-006: The desktop metaphor is the product identity](adrs/adr-006.md) — product layer (PRD phase)
- [ADR-007: v1 operator-load posture (attention + anti-clutter)](adrs/adr-007.md) — product layer (PRD phase)
- [ADR-008: Desktop state stays on the embedded KV; public surface narrows to `desktop-state`](adrs/adr-008.md) — peer-review round 1 resolution
- [ADR-009: Window snap layer on floating windows — persisted fractional zones, locally derived rendering](adrs/adr-009.md) — post-approval amendment (2026-07-20); pulls the ADR-007 snapping follow-up into scope as task_09

## Assumptions / Defaults

- Local single-operator trust model: desktop-state endpoints sit behind the same auth posture as every existing `/api` route; no new authentication is introduced.
- **Topology (peer-review B-012):** continuity spans every client connected to **one daemon** — tabs, browsers, and machines pointing at the same daemon converge; there is no daemon-to-daemon replication in this program. PRD/US-012 language matches this exactly.
- Presentation breakpoint 960px; rect-write debounce 250ms; watch buffer 256 frames; write deadline 10s; per-op mutation deadline 5s; soft open-window guidance at 12 windows (toast, no hard cap); snap constants (20px sensitivity radius, 4px drag threshold, 80ms overlay fade, 150ms zone morph, halves+quarters zone set — ADR-009) — code constants, not config.
- **Performance envelope (peer-review N-004):** with 12 windows open on a developer-class machine, pointer interactions (drag/focus/type) hold 60fps-class fluidity; desktop restore completes within ~500ms of snapshot receipt; three concurrent clients converge without visible thrash. One measured Playwright scenario guards the envelope (E2E perf journey in `_tests.md`).
- Evergreen-browser baseline (backdrop-filter, `content-visibility`); no legacy browser support.
- Multi-client concurrency resolves last-writer-wins per key by commit `seq`; no CRDT/merge semantics. Degraded-recovery policy per Safety Invariant 10.
- Session windows are unbounded in count; performance posture is minimize=unmount + per-visible-window cost.
- The `/design-system` route tree stays outside the OS shell.
- `state.csv` regeneration and scenario execution belong to the next QA cycle; this program only flags.
- The spec set (PRD, stories, tests, ADR-001..009, the market-research and Hermes-arrangement analyses) is the complete input to `cy-create-tasks`; the prototype's current working tree — including its compact block — is the visual reference bundle (the snap layer's parity contract is its constants, not a rendered reference — ADR-009).
