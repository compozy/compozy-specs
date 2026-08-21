# Developer Experience: Command Palette — OS-Grade Overhaul

Public-surface contract for the command-palette overhaul. Companion to `_spec.md` (Part II serves this surface) and `_tests.md` (E2E journeys use these exact invocations).

Scope note: the operator-facing UI surface lives in `_uiux.md`. This document covers everything typed or called: the extension manifest, CLI, HTTP/UDS, config, native tools, and the deterministic error surface.

## Golden Path

Extension author, from zero to commands in ⌘K in under a minute:

```console
$ compozy extension init notes && cd notes
Created extension scaffold at ./notes

# Declare palette contributions in extension.json (see Manifest below), then:
$ compozy extension dev .
✓ notes linked to workspace acme (dev)
✓ cmd_palette: 2 commands, 1 view registered
watching for changes…

# In the shell: ⌘K → type "cap" → "Capture note" appears under Notes,
# with its default chord ⌥⇧N shown on the row.

$ compozy cmd-palette list --source ext.notes -o json
[
  {
    "id": "ext.notes.capture",
    "title": "Capture note",
    "source": "ext.notes",
    "section": "Notes",
    "available": true,
    "bindings": ["alt+shift+KeyN"],
    "arguments": [{ "name": "title", "type": "text", "required": true }]
  },
  {
    "id": "ext.notes.recent",
    "title": "Recent notes",
    "source": "ext.notes",
    "section": "Notes",
    "available": true,
    "action": { "kind": "view", "view": "ext.notes.recent" }
  }
]

$ compozy cmd-palette invoke ext.notes.capture --arg title="Standup follow-ups"
{
  "status": "ok",
  "command": "ext.notes.capture",
  "result": { "note_id": "nt_8f2ac1", "title": "Standup follow-ups" }
}
```

## Manifest (`extension.json`)

The complete palette block an extension author writes. Loaded at enable; validated by `compozy extension build` / `validate`; hot-reloaded by `compozy extension dev`.

```jsonc
{
  "name": "notes",
  "resources": {
    "tools": {
      "capture_note": { "visibility": "model" },
      "list_recent": { "visibility": "model" }
    },
    "cmd_palette": {
      "commands": [
        {
          // Final id becomes ext.notes.capture — namespacing is automatic.
          "id": "capture",
          "title": "Capture note",
          "section": "Notes",
          "icon": "pencil", // host icon token OR a single emoji ("📝"); anything else fails validate
          "keywords": ["jot", "memo", "quick"],
          "arguments": [
            { "name": "title", "type": "text", "placeholder": "Note title", "required": true },
            { "name": "tag", "type": "dropdown", "options": ["inbox", "idea"], "required": false }
          ],
          "action": { "kind": "tool", "tool": "capture_note" },
          "default_shortcut": "alt+shift+KeyN",
          "execution": { "single_flight": true, "retry_safe": false } // optional; defaults by action kind
        },
        {
          "id": "recent",
          "title": "Recent notes",
          "section": "Notes",
          "icon": "clock",
          "action": { "kind": "view", "view": "recent" }
        },
        {
          "id": "purge",
          "title": "Purge archived notes",
          "icon": "trash",
          "action": { "kind": "tool", "tool": "purge_archived" },
          "destructive": true,
          "confirmation": {
            "title": "Purge archived notes?",
            "body": "Permanently deletes every archived note in this workspace.",
            "confirm": "Purge"
          }
        }
      ],
      "views": [
        {
          // Tier 1 — declarative: the daemon invokes this tool for the view payload.
          // Zero view code. Available to TypeScript AND Go extensions.
          "id": "recent", // final id ext.notes.recent
          "title": "Recent notes",
          "kind": "list", // list | detail | form | grid
          "source": { "tool": "list_recent" }
        },
        {
          // Tier 2 — programmable: a live view program over the extension's
          // view.provider surface (search, chips, forms, navigation, live updates).
          // TypeScript-only this cycle: a Go extension declaring "program": true
          // fails `compozy extension validate` with an actionable message.
          "id": "browser", // final id ext.notes.browser
          "title": "Browse notes",
          "kind": "list",
          "program": true
        }
      ]
    }
  }
}
```

Action union (`action.kind`): `tool` (invoke one of the extension's own tools) · `view` (push one of its declared views) · `navigate` (`{"kind":"navigate","app":"knowledge"}` — open an OS app) · `url` (`{"kind":"url","url":"https://…"}` — open externally).

### Tier 1 — declarative views (TypeScript and Go)

The view is a projection of a tool's payload into the shared frame vocabulary — the same vocabulary Tier-2 programs emit, so both tiers render identically and compose. The `source` tool must be **read-only risk class**: `compozy extension validate` rejects gated or destructive tools as view sources, so declarative views always load without policy prompts. What `source.tool` returns for `kind: "list"`:

```jsonc
{
  "view": "v1",
  "sections": [
    {
      "title": "Today",
      "rows": [
        {
          "id": "nt_8f2ac1",
          "title": "Standup follow-ups",
          "subtitle": "inbox",
          "icon": "note",
          "badge": { "label": "new", "tone": "info" },
          "actions": [
            { "title": "Open", "primary": true, "action": { "kind": "tool", "tool": "open_note", "args": { "id": "nt_8f2ac1" } } },
            { "title": "Delete", "destructive": true, "action": { "kind": "tool", "tool": "delete_note", "args": { "id": "nt_8f2ac1" } } }
          ]
        }
      ]
    }
  ],
  "chips": [{ "id": "inbox", "label": "Inbox", "count": 4 }],
  "empty": { "title": "No notes yet", "hint": "Capture one with ⌥⇧N" }
}
```

The other kinds use the same envelope with a kind-specific body:

```jsonc
// kind: "detail"
{ "view": "v1", "detail": { "markdown": "## Standup follow-ups\n- ping infra…",
    "metadata": [{ "label": "Tag", "value": "inbox" }, { "label": "Updated", "value": "2m ago" }],
    "actions": [{ "title": "Open in app", "primary": true, "action": { "kind": "navigate", "app": "knowledge" } }] } }

// kind: "form"
{ "view": "v1", "form": { "fields": [
    { "id": "title", "type": "text", "label": "Title", "required": true },
    { "id": "tag", "type": "dropdown", "label": "Tag", "options": ["inbox", "idea"] } ],
    "submit": { "title": "Save", "action": { "kind": "tool", "tool": "save_note" } } } }

// kind: "grid"
{ "view": "v1", "grid": { "sections": [{ "title": "Wallpapers", "tiles": [
    { "id": "w1", "title": "Dunes", "image": { "url": "asset://dunes.png" },
      "actions": [{ "title": "Apply", "primary": true, "action": { "kind": "tool", "tool": "apply_wallpaper", "args": { "id": "w1" } } }] } ] }] } }
```

The tool behind a Tier-1 view is ordinary extension code — both languages:

```go
// Go — sdk/go
sdk.RegisterTool(ext, sdk.Tool[ListRecentInput]{
    Name: "list_recent",
    Handler: func(ctx sdk.ToolContext, in ListRecentInput) (any, error) {
        notes := loadRecent(in.Limit)
        rows := make([]map[string]any, 0, len(notes))
        for _, n := range notes {
            rows = append(rows, map[string]any{
                "id": n.ID, "title": n.Title, "subtitle": n.Tag, "icon": "note",
                "actions": []map[string]any{{"title": "Open", "primary": true,
                    "action": map[string]any{"kind": "tool", "tool": "open_note", "args": map[string]any{"id": n.ID}}}},
            })
        }
        return map[string]any{"view": "v1", "sections": []map[string]any{{"title": "Today", "rows": rows}}}, nil
    },
})
```

```ts
// TypeScript — @compozy/extension-sdk
extension.tool("list_recent", { visibility: "model" }, async (ctx) => {
  const notes = await loadRecent();
  return {
    view: "v1",
    sections: [{ title: "Today", rows: notes.map((n) => ({
      id: n.id, title: n.title, subtitle: n.tag, icon: "note",
      actions: [{ title: "Open", primary: true, action: { kind: "tool", tool: "open_note", args: { id: n.id } } }],
    })) }],
  };
});
```

### Tier 2 — view programs (TypeScript)

A programmable view is a live React program running **in the extension's subprocess** (never in the operator's browser). `@compozy/extension-react` ships the palette component kit and hooks; a reconciler turns renders into frames/patches on the wire. The authoring model is Raycast's:

```tsx
// src/views/browser.tsx
import {
  List, Detail, Form, ActionPanel, Action,
  useNavigation, useCachedPromise, showToast,
} from "@compozy/extension-react";

export function NotesBrowser() {
  const [query, setQuery] = useState("");
  const [chip, setChip] = useState<string | null>(null);
  const { data, isLoading, mutate } = useCachedPromise(searchNotes, [query, chip]);
  const { push } = useNavigation();

  return (
    <List
      isLoading={isLoading}
      onSearchTextChange={setQuery}   // presence = extension-authoritative search; host stops filtering
      throttle                         // host-side debounce; keystrokes coalesce
      chips={[{ id: "inbox", label: "Inbox", count: data?.counts.inbox },
              { id: "idea",  label: "Ideas", count: data?.counts.idea }]}
      activeChip={chip}
      onChipToggle={setChip}
    >
      {data?.notes.map((n) => (
        <List.Item key={n.id} id={n.id} title={n.title} subtitle={n.tag} icon="note"
          badge={n.fresh ? { label: "new", tone: "info" } : undefined}
          actions={
            <ActionPanel>
              <Action.Push title="Open" target={<NoteDetail id={n.id} />} />
              <Action.Push title="Edit" target={<EditNote note={n} onSaved={() => mutate()} />} />
              <Action title="Delete" style="destructive"
                confirmation={{ title: "Delete note?", body: `Permanently deletes “${n.title}”.`, confirm: "Delete" }}
                onAction={async () => { await deleteNote(n.id); await mutate();
                  await showToast({ tone: "success", message: "Note deleted" }); }} />
            </ActionPanel>
          } />
      ))}
    </List>
  );
}

function NoteDetail({ id }: { id: string }) {
  const { data, isLoading } = useCachedPromise(loadNote, [id]);
  return <Detail isLoading={isLoading} markdown={data?.body}
    metadata={[{ label: "Tag", value: data?.tag }, { label: "Updated", value: data?.updatedAt }]} />;
}

function EditNote({ note, onSaved }: { note: Note; onSaved: () => void }) {
  const { pop } = useNavigation();
  return (
    <Form onSubmit={async (values) => { await saveNote(note.id, values); onSaved(); pop(); }}>
      <Form.TextField id="title" label="Title" defaultValue={note.title} required />
      <Form.Dropdown id="tag" label="Tag" defaultValue={note.tag} options={["inbox", "idea"]} />
    </Form>
  );
}
```

```ts
// src/index.ts — wiring the program into the extension
import { Extension } from "@compozy/extension-sdk";
import { registerReactViews } from "@compozy/extension-react";
import { NotesBrowser } from "./views/browser";

const extension = new Extension({ name: "notes" });
registerReactViews(extension, { browser: NotesBrowser }); // ids match manifest views[].id
await extension.start();
```

What the runtime guarantees (and expects) — the contract behind the sugar:

- **Typing is local.** Search echo, selection, scroll, and chip highlights render in 0 ms host-side; your `onSearchTextChange` fires host-throttled. Return your result sets with `complete: true` when they are the full universe — the host then filters locally between your updates.
- **One live session per attached client.** Two windows open on your view = two independent sessions of your component (own search, own navigation). Sessions end when the palette closes; an extension restart drops sessions and the host reopens fresh.
- **Slow is visible, never fatal — until it is repeated.** Past ~150 ms the host shows your previous rows plus a busy indicator; past 3 s without a frame the view degrades with a retry; three misses in a row circuit-breaks the view until re-invoked. A crash shows the unavailable frame and never takes the palette down.
- **Don't block the event loop.** Your handlers share the process event loop with the daemon's health probe — synchronous CPU beyond the budget can crash-loop your own extension. Chunk heavy work (the SDK warns when a handler exceeds the yield budget).
- **Confirmation and policy still apply.** `style: "destructive"` requires a declared `confirmation` (kit validation fails without it) and renders the palette's confirmation step. Actions that carry a declared target (`Action.Push`, `Action.Open`, `Action.OpenApp`, `Action.CopyToClipboard`, tool-backed actions) are dispatched by the host through the same policy gates as commands — before your code hears about them. Bare `onAction` handlers are view-local: view state plus your own domain logic under your extension's existing grants; the daemon treats them as opaque and grants them nothing new.
- **Effects are a closed union, not open host access.** `showToast`, `Action.CopyToClipboard`, `Action.Open`, `Action.OpenApp`, and file picking compile to frame-visible effects the host executes under the same rules as command actions (URL/app targets included). Everything else your handlers do stays inside your extension process under its existing grants — there is no other door into the daemon or the operator's machine.

#### The React kit at a glance

The kit ships as its **own workspace package** — **`@compozy/extension-react`**, at `sdk/react/` — depending on `@compozy/extension-sdk` (transport, provide-surface registrar, view sessions) and on the generated frame typings. `react`/`react-reconciler` are ordinary dependencies of this package only: an extension without program views never installs it. In-repo everything versions together, and the repo gates keep the pairing honest — `make codegen` regenerates the vocabulary typings and `codegen-check` + the Bun typecheck lane fail the moment the kit lags a vocabulary change. Every component compiles to the same frame vocabulary Tier-1 payloads use; handler-valued props (marked ▸) become stable opaque handler ids on the wire — functions never cross.

| Component | Renders as | Key props |
| --- | --- | --- |
| `<List>` | list frame | `isLoading` · `searchText` (controlled) · `searchBarPlaceholder` · `throttle` · `filtering` (override; auto-false when ▸`onSearchTextChange` present) · `complete` (host filters locally when `true`) · `chips` · `activeChip` · `pagination{hasMore, pageSize, ▸onLoadMore}` · ▸`onSearchTextChange` · ▸`onChipToggle` · ▸`onSelectionChange` |
| `<List.Section>` | row section | `title` |
| `<List.Item>` | row | `id` (required, stable) · `title` · `subtitle` · `icon` (token \| emoji) · `badge{label, tone}` · `accessories[]` · `keywords[]` · `detail` (`<List.Item.Detail>`) · `actions` (`<ActionPanel>`) |
| `<List.Item.Detail>` | split preview pane | `isLoading` · `markdown` · `metadata[{label, value}]` |
| `<List.EmptyView>` | honest empty state | `title` · `hint` · `icon` |
| `<Detail>` | detail frame | `isLoading` · `markdown` · `metadata[]` · `actions` |
| `<Grid>` / `<Grid.Section>` / `<Grid.Item>` | grid frame | Grid: List-parity search/chip props + `columns`; Item: `id` · `title` · `image{url \| token \| emoji}` · `badge` · `actions` |
| `<Form>` | form frame | ▸`onSubmit(values)` · `validation` · children fields |
| `<Form.TextField>` · `.PasswordField` · `.TextArea` · `.Checkbox` · `.Dropdown` (+ `.Dropdown.Item`) · `.FilePicker` | typed fields | `id` · `label` · `defaultValue` · `placeholder` · `required` · `error` · ▸`onChange` · ▸`onBlur` (Dropdown: `options` \| items; FilePicker: `directories`) |
| `<ActionPanel>` / `<ActionPanel.Section>` | ⌘K action panel | section `title` |
| `<Action>` | panel action | `title` · `icon` · `shortcut` · `style: "default" \| "destructive"` · `confirmation{title, body, confirm}` · ▸`onAction` |
| `<Action.Push>` | push a component | `title` · `target={<Component …props/>}` (props baked into the element) |
| `<Action.SubmitForm>` | submit enclosing form | `title` |
| `<Action.Open>` | open external URL | `title` · `url` |
| `<Action.OpenApp>` | navigate to an OS app | `title` · `app` |
| `<Action.CopyToClipboard>` | host copy effect | `title` · `content` |

| Hook / utility | Contract |
| --- | --- |
| `useNavigation()` | `{ push(<Component/>), pop(), popToRoot() }` — parent frames stay alive; ⌫/Esc pop host-side |
| `usePromise(fn, deps)` | `{ data, isLoading, error, mutate }` — `mutate` supports `optimisticUpdate`/`rollbackOnError` |
| `useCachedPromise(fn, deps, {keepPreviousData})` | `usePromise` + stale-while-revalidate + cursor pagination (`loadMore`) |
| `useCachedState(key, initial)` | state persisted per extension across sessions |
| `showToast({tone, message})` | host toast; survives view close |

Anything beyond this closed kit (arbitrary layout, HTML, custom widgets) is out of the vocabulary by design — new components require a vocabulary revision (ADR-004 rule).

## CLI

### `compozy cmd-palette list`

```console
$ compozy cmd-palette list
COMMANDS (workspace: acme)
  Window
    window.close            Close window                    ⌘W
    window.zoom             Zoom window                     ⌃⌥Return
    …
  Notes (ext.notes)
    ext.notes.capture       Capture note                    ⌥⇧N
    ext.notes.recent        Recent notes
  … 142 commands · 96 available · use -o json for full detail
```

```console
$ compozy cmd-palette list --available=false -o jsonl
{"id":"window.merge.all","title":"Merge all windows","available":false,"reason":"needs two windows on this desktop"}
{"id":"ext.notes.capture","title":"Capture note","available":false,"reason":"extension notes is unhealthy (crash loop)"}
```

Flags: `--workspace <name>` · `--source core|ext.<name>` · `--available[=bool]` · `--client <id>` (resolve client-context availability against that attachment; omitted → such commands report `requires an attached shell`) · `-o json|jsonl|toon`. Exit 0 (even when empty).

### `compozy cmd-palette inspect <id>`

```console
$ compozy cmd-palette inspect ext.notes.purge -o json
{
  "id": "ext.notes.purge",
  "title": "Purge archived notes",
  "source": "ext.notes",
  "available": true,
  "destructive": true,
  "confirmation": { "title": "Purge archived notes?", "confirm": "Purge" },
  "action": { "kind": "tool", "tool": "ext__notes__purge_archived" },
  "execution": { "single_flight": true, "retry_safe": false },
  "bindings": [],
  "alias": null,
  "risk": "elevated",
  "approval_required": true
}
```

Exit 0; exit 1 with `command not found: <id>` on unknown id.

### `compozy cmd-palette invoke <id>`

```console
$ compozy cmd-palette invoke ext.notes.capture --arg title="Standup follow-ups" --arg tag=inbox
{ "status": "ok", "command": "ext.notes.capture", "result": { "note_id": "nt_9a01d2" } }

$ compozy cmd-palette invoke window.close
Error: no attached shell client — window.close changes UI state and needs an open Compozy shell.
(exit 1)

$ compozy cmd-palette invoke ext.notes.purge
{ "status": "approval_pending", "approval_id": "apr_55e0c9", "message": "destructive command requires approval" }

$ compozy cmd-palette invoke ext.notes.capture
Error: invalid arguments — missing required "title".
(exit 2)
```

Flags: `--arg k=v` (repeatable) · `--workspace <name>` · `--client <id>` (UI-effecting commands; omit when exactly one client is attached) · `-o json`. Exit 0 ok/approval-pending · 1 runtime failure · 2 validation.

```console
$ compozy cmd-palette clients -o json
[ { "client_id": "cl_7f21", "kind": "shell", "workspace": "acme", "attached_at": "2026-08-18T00:12:03Z" } ]

$ compozy cmd-palette invoke window.close        # two clients attached, no --client
Error: multiple attached clients — pass --client (cl_7f21 shell, cl_a09c browser).
(exit 1)
```

### `compozy cmd-palette personalization`

```console
$ compozy cmd-palette personalization show -o json
{ "workspace": "acme", "pins": ["session.new"], "recents": 18, "frecency_entries": 64, "query_associations": 21 }

$ compozy cmd-palette personalization reset --workspace acme
Reset palette personalization for workspace acme (pins, recents, frecency, query learning).
```

### `compozy cmd-palette bind` · `unbind` · `alias` · `bindings`

```console
$ compozy cmd-palette bind ext.notes.capture "meta+shift+KeyN"
Error: shortcut conflict — meta+shift+KeyN is used by "session.new". Re-run with --overwrite to take it.
(exit 1)

$ compozy cmd-palette bind ext.notes.capture "meta+shift+KeyN" --overwrite -o json
{ "status": "ok", "bound": ["meta+shift+KeyN"], "unbound_owner": "session.new" }

$ compozy cmd-palette alias set ext.notes.capture cap -o json
{ "status": "ok", "alias": "cap" }

$ compozy cmd-palette alias clear ext.notes.capture && compozy cmd-palette unbind ext.notes.capture
{ "status": "ok" }
{ "status": "ok" }

$ compozy cmd-palette bindings -o json
{
  "effective": { "ext.notes.capture": ["alt+shift+KeyN"], "session.new": ["meta+KeyN"] },
  "aliases": { "ext.notes.capture": "cap" },
  "dormant_defaults": [ { "command": "ext.other.jump", "chord": "meta+KeyJ", "conflict_with": "window.tab.jump.1" } ],
  "conflicts": []
}
```

```console
$ compozy cmd-palette pin session.new && compozy cmd-palette unpin session.new
{ "status": "ok", "pinned": true }
{ "status": "ok", "pinned": false }

$ compozy cmd-palette alias set ext.other.jump cap
Error: alias conflict — "cap" is owned by "ext.notes.capture". Re-run with --overwrite to take it.
(exit 1)
```

Every mutation is atomic through its daemon path (aliases/bindings via the settings PATCH; pins via `PUT|DELETE /api/cmd-palette/pins/{id}`) and returns the same structured errors as HTTP (`shortcut_conflict`/`alias_conflict` naming the owner, `invalid_alias` with the grammar). Aliases are unique per workspace; `--overwrite` (CLI) / `"overwrite": true` (PATCH) transfers ownership atomically, clearing the loser. Exit 0 ok · 1 conflict/runtime · 2 validation.

## HTTP / UDS API

Identical routes on both transports.

### `GET /api/cmd-palette/commands?workspace=acme&client=cl_7f21`

```json
200
{
  "commands": [
    {
      "id": "window.close",
      "title": "Close window",
      "source": "core",
      "section": "Window",
      "icon": "x-square",
      "available": true,
      "bindings": ["meta+KeyW"],
      "alias": null,
      "destructive": false,
      "arguments": []
    },
    {
      "id": "ext.notes.capture",
      "title": "Capture note",
      "source": "ext.notes",
      "section": "Notes",
      "icon": "pencil",
      "available": true,
      "bindings": ["alt+shift+KeyN"],
      "arguments": [{ "name": "title", "type": "text", "required": true }]
    }
  ],
  "sources": [
    { "source": "core", "status": "healthy" },
    { "source": "ext.notes", "status": "healthy" }
  ],
  "catalog_revision": "cr_01J9…"
}
```

`client` selects whose context resolves client-context availability (here: `window.close` is `available:true` against `cl_7f21`'s snapshot). **Listing never auto-selects a client**: without `client`, client-context commands list as `available:false, reason:"requires an attached shell"` — with zero, one, or many attachments alike. Auto-selection (exactly one attached) applies only to invoke.

### `POST /api/cmd-palette/commands/{id}/invoke`

```json
{ "workspace": "acme", "args": { "title": "Standup follow-ups" } }
```

```json
200 { "status": "ok", "result": { "note_id": "nt_9a01d2" } }
202 { "status": "approval_pending", "approval_id": "apr_55e0c9" }
404 { "error": "command_not_found", "message": "unknown command: ext.notes.captur" }
409 { "error": "already_running", "message": "ext.notes.purge is already in flight" }
412 { "error": "command_unavailable", "reason": "needs two windows on this desktop" }
412 { "error": "no_attached_shell", "message": "window.close changes UI state and needs an open Compozy shell" }
422 { "error": "invalid_arguments", "fields": { "title": "required" } }
```

### `GET /api/cmd-palette/personalization?workspace=acme` · `DELETE /api/cmd-palette/personalization?workspace=acme`

```json
200 { "workspace": "acme", "pins": ["session.new"], "recents": 18, "frecency_entries": 64, "query_associations": 21 }
200 { "status": "reset" }
```

### `GET /api/cmd-palette/rank-signals?workspace=acme`

The single client-side scorer's input projection — session-memory only, never persisted by the browser:

```json
200
{
  "weights": { "version": 1, "frecency_half_life_days": 30, "query_half_life_days": 14, "deadband": 12, "fallback_weak_match_threshold": 120 },
  "usage": [ { "command_id": "session.new", "weight": 141.2, "last_used_at": 1765995000123 } ],
  "query_hits": [ { "query": "gh", "command_id": "ext.notes.capture", "weight": 210.4 } ],
  "pins": ["session.new"],
  "revision": "ps_01J9…"
}
```

### Remaining routes — frozen shapes

```jsonc
// Pins (idempotent)
PUT    /api/cmd-palette/pins/session.new?workspace=acme        → 200 { "pinned": true }
DELETE /api/cmd-palette/pins/session.new?workspace=acme        → 200 { "pinned": false }

// Usage report (client-executed commands; normalized query only — never argument values)
POST /api/cmd-palette/usage    { "workspace": "acme", "command_id": "session.new", "query": "ns" } → 204

// Palette settings section (fallback + personalization switch)
GET  /api/settings/cmd-palette  → 200 { "fallback_agent_enabled": true, "personalization": true }
PATCH /api/settings/cmd-palette { "fallback_agent_enabled": false } → 200 (same shape) · 422 invalid value

// Declarative (Tier-1) view + live refresh
GET /api/cmd-palette/views/ext.notes.recent?workspace=acme
    → 200 CmdPaletteViewEnvelope { view_id, title, kind, revision, stream_epoch, payload: ViewPayload }
GET /api/cmd-palette/views/ext.notes.recent/stream?workspace=&after=&stream_epoch=
    → SSE of revision-fenced patches · 400 when after>0 without stream_epoch · full resync on fence mismatch

// Programmable (Tier-2) view sessions — self-originated calls carry X-Compozy-Client-Token
POST /api/cmd-palette/views/ext.notes.browser/open  { "workspace": "acme", "args": {} }
    → 200 { "view_session": "vs_02ab", "stream_token": "vst_9c1f", "first_frame": { …ViewFrame } }
    → 401 invalid/missing attachment token · 404 unknown view · 422 program-required (Tier-1 id)
GET  /api/cmd-palette/view-sessions/vs_02ab/stream?token=vst_9c1f   → SSE of ViewFrames (session-scoped)
POST /api/cmd-palette/view-sessions/vs_02ab/events  { "handler": "h_71", "args": ["meet", 4], "seq": 5, "revision": "vr_11", "ack_effects": ["ef_3"] }
    → 202 · 409 view_busy (caps) · 410 session gone (closed/restart) · 403 ownership
DELETE /api/cmd-palette/view-sessions/vs_02ab                        → 200 (idempotent)

// Async approvals (tools-owned, generic — the 202's approval_id resolves here)
GET  /api/tools/approvals/apr_55e0c9
    → 200 { "approval_status": "approved", "execution_status": "completed", "result": { … } }
    → 200 { "approval_status": "pending", "expires_at": "2026-08-18T02:10:00Z" }
    → 200 { "approval_status": "approved", "execution_status": "uncertain" }   // crash window — never silently retried
POST /api/tools/approvals/apr_55e0c9/cancel                          → 200 { "approval_status": "canceled" } · 409 already terminal
```

All of the above are registered on HTTP **and** UDS with OpenAPI/generated-TS co-ship; `GET /clients` and `GET /rank-signals` are frozen above. CLI parity: pins/aliases/bindings verbs shown earlier; approvals via `compozy approvals show|cancel <id>`.

### Bindings and aliases — existing settings surface, extended

`GET /api/settings/window-manager` now returns every registry command (core + extension) in `effective_shortcuts`, plus `aliases` and `extension_defaults` (bound + dormant with conflict reasons). `PATCH` accepts the same shapes:

```json
PATCH /api/settings/window-manager
{ "shortcuts": { "ext.notes.capture": ["meta+shift+KeyN"] }, "aliases": { "ext.notes.capture": "cap" } }

200 { "effective_shortcuts": { "ext.notes.capture": ["meta+shift+KeyN"] }, "aliases": { "ext.notes.capture": "cap" } }
409 { "error": "shortcut_conflict", "owner": "session.new", "chord": "meta+KeyN" }
422 { "error": "invalid_alias", "message": "aliases are 1-32 chars, no whitespace" }
```

### SSE

`cmd_palette.catalog.changed { workspace, catalog_revision }` — emitted on extension enable/disable/reload/health transitions and registry changes; clients refetch `GET /api/cmd-palette/commands`.

## config.toml

```toml
[cmd_palette]
fallback_targets = ["agent"]   # ordered; first target is the ⏎ default on the fallback row
personalization  = true        # master switch: frecency + query learning + recents recording

[window_manager.shortcuts]
"palette.open"          = ["meta+KeyK", "meta+shift+KeyP"]   # unchanged
"palette.view.sessions" = ["meta+KeyE"]                       # unchanged

[window_manager.global_shortcuts]                             # OS-global; desktop shell only
"palette.summon.global" = "meta+shift+Space"                  # default entry — any command may appear here
"ext.notes.capture"     = "meta+alt+KeyN"                     # e.g. a per-command global hotkey
```

- `cmd_palette.fallback_targets` — which delegation targets the fallback row offers, in order. v1 ships `"agent"` (new session with the workspace default agent).
- `cmd_palette.personalization` — `false` stops recording and neutralizes personal ranking signals (existing data kept until reset).
- `[window_manager.global_shortcuts]` — daemon-owned intended OS-global bindings for any command; the shell registers them and acknowledges per-chord status (`registered` / `failed_in_use` / `failed_permission`), and a chord displays as active only once confirmed. In-browser installs ignore the section and Settings shows why. Mutate via `compozy cmd-palette bind <id> <chord> --global` / `unbind <id> --global` or the settings PATCH.

## Native Tools

### `compozy__cmd_palette_list`

```json
{ "workspace": "acme", "source": "ext.notes", "client": "cl_7f21" }
```

`client` is optional and mirrors CLI `--client`: it resolves client-context availability against that attachment (control-plane authorization — no attachment token involved).

```json
{ "commands": [ { "id": "ext.notes.capture", "title": "Capture note", "available": true, "arguments": [...] } ] }
```

### `compozy__cmd_palette_invoke`

```json
{ "id": "ext.notes.capture", "workspace": "acme", "args": { "title": "Standup follow-ups" }, "client": "cl_7f21" }
```

`client` is optional with the same resolution as CLI/HTTP: one attached → auto; several → structured `multiple_clients` error listing ids; UI-effecting with none → `no_attached_shell`.

```json
{ "status": "ok", "result": { "note_id": "nt_9a01d2" } }
```

Gates: `compozy__cmd_palette_invoke` carries the invoked command's effective risk — destructive/approval-gated commands return `approval_pending` exactly like the HTTP surface; UI-effecting commands require an attached shell and fail with `no_attached_shell` otherwise.

## Errors

| Condition | Surface behavior | Points to |
| --- | --- | --- |
| Unknown command id | 404 / exit 1 `command_not_found` | `compozy cmd-palette list` |
| Args fail declared schema | 422 / exit 2 `invalid_arguments` with per-field reasons | `compozy cmd-palette inspect <id>` |
| Command unavailable in context | 412 `command_unavailable` + the same `reason` string the UI shows | context (reason text) |
| UI command, no shell attached | 412 / exit 1 `no_attached_shell` | open the shell |
| UI command, several clients attached, no `client` | 409 / exit 1 `multiple_clients` listing ids | `compozy cmd-palette clients` / `--client` |
| Non-idempotent command already in flight | 409 `already_running` | wait / status |
| Destructive or approval-gated | 202 `approval_pending` + `approval_id` | approvals surface |
| Chord conflict on bind | 409 `shortcut_conflict` naming the owning command | settings shortcut table |
| Invalid alias grammar | 422 `invalid_alias` with the rule | — |
| Alias already owned | 409 `alias_conflict` naming the owning command; explicit overwrite transfers | `--overwrite` / `"overwrite": true` |
| Manifest palette block invalid | `compozy extension build/validate` fails naming the field: `cmd_palette.commands[2].action.tool: unknown tool "purge_archved"` | fix manifest |
| Extension unhealthy | commands listed `available:false` with `reason:"extension <name> is unhealthy (crash loop)"` | `compozy extension logs` |
| Runtime (daemon) unreachable from UI | rows disabled with reason `"runtime unavailable"`; availability-exempt commands keep working | reconnect |
| View program misses the 3 s ack | view enters `degraded`: last-good rows stay, inline retry affordance | `compozy extension logs` |
| View program misses 3 acks in a row | view circuit-breaks ("view disabled until reopened"); other views unaffected | re-invoke / fix program |
| View program crashes mid-session | "view unavailable" frame naming the extension; palette stays usable; session reopens fresh | `compozy extension logs` |
| `program: true` in a Go extension | `compozy extension validate` fails: `views[1].program: view programs require a TypeScript extension this release` | use a Tier-1 `source` view |
