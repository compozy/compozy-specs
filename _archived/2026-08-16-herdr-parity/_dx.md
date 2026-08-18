# Developer Experience: Herdr Parity (Session Attention · Orchestration DX · Shortcuts v2)

Public-surface contract for the herdr-parity program. Companion to `_spec.md` (Part II serves this surface) and `_tests.md` (E2E journeys use these exact invocations). Everything below is written as if shipped.

## Golden Path

Delegate work, keep working, get pulled in only when needed — no polling, no shell babysitting.

```console
$ compozy spawn --agent researcher --ttl-seconds 3600 --prompt-overlay "Audit our HTTP client wrappers"
SESSION   ses_01jm2e6r9k
AGENT     researcher
ROLE      worker
TTL       3600s
WAKE      on-settle (default)

# ...you keep working in your own session/terminal.
# The child hits a decision point and asks a question. Your shell session is
# woken with a synthetic prompt (wake-on-settle); in the web shell the session
# dot flips to needs-input, the bell counts it, a toast + sound fire.

$ compozy session wait ses_01jm2e6r9k --until waiting-for-input,waiting-for-auth,stopped,failed
STATE     waiting-for-input
WAITED    2m41s
SESSION   ses_01jm2e6r9k

$ compozy session clarify pending ses_01jm2e6r9k
REQUEST   clr_7f2a
QUESTION  Should I include the vendored packages in the audit?
CHOICES   1. Yes, include vendored  2. No, first-party only

$ compozy session clarify answer ses_01jm2e6r9k clr_7f2a --choice 2
ANSWERED  clr_7f2a (choice 2)

$ compozy session wait ses_01jm2e6r9k
STATE     stopped
WAITED    6m03s
SESSION   ses_01jm2e6r9k
```

One command started the worker, one command blocked exactly until it needed a human, and the answer unblocked it — the same flow an orchestrator agent runs with native tools, no shell required.

## CLI

### `compozy session wait` — generalized

```console
$ compozy session wait ses_01jm2e6r9k --until waiting-for-input,idle --timeout 120s
STATE     idle
WAITED    38s
SESSION   ses_01jm2e6r9k
```

```console
$ compozy session wait ses_01jm2e6r9k --until idle --timeout 5s -o json
{"session_id":"ses_01jm2e6r9k","outcome":"timeout","waited_ms":5000,"until":["idle"]}
```

- `--until <state,...>` — target states. Vocabulary: `waiting-for-input`, `waiting-for-auth`, `idle`, `done`, `running`, `stopped`, `failed`, `hung`, `unhealthy`. Omitted → settled set `waiting-for-input,waiting-for-auth,idle,stopped,failed`. `done` satisfies `idle`.
- `--timeout <duration>` — default `300s`. `--unbounded` waits forever (mutually exclusive with `--timeout`); under the hood the CLI transparently repeats bounded server-side waits, resuming the same server registration each time (`resume_id`) — gapless, so a state entered and left between requests is never missed; the daemon itself never holds an unbounded wait.
- Already in a target state → returns immediately with that state.
- Exit codes: `0` state reached · `75` timeout · `69` session gone/daemon unavailable · `65` invalid `--until` token.

```console
$ compozy session wait ses_01jm2e6r9k --until sleeping
Error: unknown session state "sleeping" (valid: waiting-for-input, waiting-for-auth, idle, done, running, stopped, failed, hung, unhealthy)
# exit 65
```

### `compozy session list` — attention filters and Show all

```console
$ compozy session list --attention
SESSION          AGENT       BADGE               AGE
ses_01jm2e6r9k   researcher  waiting-for-input   3m
ses_01jm1zqv04   reviewer    waiting-for-auth    11m
ses_01jkzy8ne2   builder     failed              26m
```

```console
$ compozy session list --badge done --all-workspaces -o json
{"sessions":[{"id":"ses_01jm0x3tx7","workspace_id":"ws_api","agent_name":"tester","badge":"done", "...":"..."}]}
```

- `--attention` — needs-you class only (`waiting-for-input` + `waiting-for-auth` + `failed`).
- `--badge <state,...>` — exact-state filter, same vocabulary as `wait --until`.
- `--all-workspaces` — every workspace, rows carry `workspace_id` (human output adds a WORKSPACE column). Operator affordance; native `session_list` stays workspace-scoped.
- Empty result → empty table / `{"sessions":[]}`, exit `0`.

### `compozy session prompt-cancel` — new verb for the existing cancel route

```console
$ compozy session prompt-cancel ses_01jm2e6r9k
CANCELED  turn tur_8c1d
SESSION   ses_01jm2e6r9k

$ compozy session prompt-cancel ses_01jm2e6r9k
NOTHING IN FLIGHT
SESSION   ses_01jm2e6r9k
```

- Exit codes: `0` canceled · `66` nothing in flight (session survives both) · standard errors otherwise.
- `-o json`: `{"session_id":"…","outcome":"canceled","turn_id":"tur_8c1d"}` / `{"outcome":"nothing-in-flight"}`.

### `compozy notify` — send the operator a notification

```console
$ compozy notify "Deps audit done" --body "3 findings, 1 high severity"
OUTCOME   delivered
```

- Runs under agent identity (same resolution as other agent-aware verbs). Outcomes are daemon-provable: `delivered` (published to at least one live operator client), `no-client`, `muted-workspace`, `rate-limited`. The daemon never claims a human saw it; focus suppression is client-side rendering.
- Title ≤ 80 chars, body ≤ 240 chars, sanitized; 1 notification per second per session.

### Notification settings round-trip

```console
$ compozy config get attention
[attention]
toasts = true
sound = true
system = false
muted_workspaces = []

$ compozy config set attention.sound false
Updated attention.sound = false (applies live)

$ compozy config set attention.muted_workspaces '["ws_0123456789abcdef"]'
Updated attention.muted_workspaces (applies live)
```

### Shortcut arrays, ranges, and the preset

```console
$ compozy config set window_manager.shortcuts."window.focus.left" '["control+ArrowLeft","alt+KeyH"]'
Updated window_manager.shortcuts.window.focus.left

$ compozy config set window_manager.shortcuts."desktop.switch" "meta+control+Digit1..9"
Updated window_manager.shortcuts.desktop.switch (expands to desktop.switch.1..9)

$ compozy config set window_manager.shortcuts."session.cycle.next" "control+alt+KeyJ"
Error: chord "control+alt+KeyJ" already bound to window.tile.bottom-left
# exit 78
```

Grammar notes: each action accepts a chord string or an array; array members of indexed families may themselves be range strings (each expands, and validation runs on the full expansion); an empty binding (`""` or `[]`) disables the action's shortcut.

The terminal preset is applied from Settings → Shortcuts (one click, preview + revert); its full binding block is documented on the shortcuts docs page as pasteable `[window_manager.shortcuts]` lines, so terminal users and agents can apply it via config file or `config set` line by line.

## HTTP / UDS API

Same routes and bodies on both transports.

### `POST /api/workspaces/{workspace_id}/sessions/{session_id}/wait`

```json
// request
{ "until": ["waiting-for-input", "idle"], "timeout_ms": 120000 }

// 200 — state reached
{ "session_id": "ses_01jm2e6r9k", "outcome": "state-reached", "state": "idle", "waited_ms": 38000 }

// 200 — timeout (bounded long-poll; timeout is a result, not a transport error)
{ "session_id": "ses_01jm2e6r9k", "outcome": "timeout", "waited_ms": 120000, "until": ["waiting-for-input", "idle"] }

// 404 — unknown session · 410 — session deleted mid-wait
{ "error": { "code": "session_gone", "message": "session ses_01jm2e6r9k no longer exists" } }
```

`timeout_ms` required over the API (max `1800000`); omitted/oversized → `422`.

### `POST /api/agent/notify` (agent identity required)

```json
// request
{ "title": "Deps audit done", "body": "3 findings, 1 high severity" }

// 200 — published to at least one live operator client
{ "outcome": "delivered" }
// 200 — no live operator client connected
{ "outcome": "no-client" }
// 200 — rate limited (deterministic result, not an error)
{ "outcome": "rate-limited", "retry_after_ms": 1000 }

// 422 — title over 80 chars
{ "error": { "code": "invalid_notification", "message": "title exceeds 80 characters" } }
```

### `POST /api/workspaces/{workspace_id}/sessions/{session_id}/presence`

Operator-only, per-client presence lease — how the daemon knows a session is being viewed. Acquire when the session's window gains focus with the app visible; renew every 5 seconds (lease TTL 15s); release on blur. Each client owns its own lease (`lease_id`), so one client can never release another's; while any lease is live the daemon marks the session seen continuously, and a turn that settles under a live lease never becomes `done`.

```json
// acquire
{ "visible": true }
// 200
{ "lease_id": "prl_01jm5q2c8t" }

// renew / release (lease_id required)
{ "visible": true, "lease_id": "prl_01jm5q2c8t" }   // 204
{ "visible": false, "lease_id": "prl_01jm5q2c8t" }  // 204

// 403 — requests carrying agent identity are denied (operator affordance only)
{ "error": { "code": "agent_scope_denied", "message": "presence is an operator surface" } }
```

### `GET /api/workspaces/{workspace_id}/sessions/{session_id}/interactions`

Sanitized pending-interaction discovery — the canonical record behind `waiting-for-input`/`waiting-for-auth`, restart-durable and listable (`?status=pending` default; both kinds). The same projection embeds in session detail/status payloads, so `compozy__session_status` gives agents the ids they need to approve/answer after a restart.

```json
// 200
{ "interactions": [
  { "interaction_id": "int_01jm5r7d3k", "kind": "clarify", "provider_request_id": "clr_7f2a",
    "title": "Should I include the vendored packages in the audit?",
    "choices": ["Yes, include vendored", "No, first-party only"],
    "status": "orphaned", "created_at": "2026-08-15T18:02:41Z" }
] }
```

```console
$ compozy session interactions ses_01jm2e6r9k
KIND      REQUEST   STATUS     TITLE
clarify   clr_7f2a  orphaned   Should I include the vendored packages in the audit?
```

Resolving an orphaned interaction records the decision and enqueues it as a queued prompt in one atomic step — if the session's input queue is full, nothing changes and the caller gets the retryable `queue-full` outcome.

### `GET /api/sessions/attention-summary`

Operator-only exact cross-workspace counts — the bell badge and tab title consume this projection (never row pages), so counts stay exact past any page size.

```json
// 200
{ "needs_you": 3, "finished": 5,
  "by_workspace": [ { "workspace_id": "ws_api", "needs_you": 2, "finished": 1 },
                    { "workspace_id": "ws_web", "needs_you": 1, "finished": 4 } ] }
```

```console
$ compozy session list --summary
NEEDS YOU  3
FINISHED   5
ws_api     needs-you 2 · finished 1
ws_web     needs-you 1 · finished 4
```

### `POST /api/agent/spawn` — gains `notify_creator`

```json
// request (new field; default true)
{ "agent_name": "researcher", "ttl_seconds": 3600, "notify_creator": true, "prompt_overlay": "Audit our HTTP client wrappers" }
```

CLI parity:

```console
$ compozy spawn --agent researcher --ttl-seconds 3600 --no-notify-creator
SESSION   ses_01jm4a8x2p
WAKE      off
```

When the child stops, fails, or enters a needs-you state, the parent receives a synthetic prompt:

```
Session wake: child session ses_01jm2e6r9k (researcher) is waiting-for-input.
Reason: needs_attention. Question: "Should I include the vendored packages in the audit?"
Use compozy__session_clarify_answer or compozy__session_events to act.
```

### Payload changes on existing routes

- `SessionPayload.badge` vocabulary gains `waiting-for-input` and `done`, and the payload gains `attention_changed_at` (the restart-stable ordering key for bell rows and attention-first sorts) — list, detail, status, catalog pages, aggregates `by_badge`; session detail/status payloads embed the sanitized pending interactions.
- The catalog stream (`session_catalog_changed`) gains two named companions: `session_attention_changed` (transition edges `{session_id, workspace_id, from, to, class, at}` — used only to fire toasts/sound/system alerts) and `operator_notification` (sanitized `{notification_id, session_id, workspace_id, title, body, at}` — the transport behind agent notifications). State always comes from refetch on the normal wake, which also fires on seen-clears.
- `GET /api/sessions` (cross-workspace catalog) serves the CLI `--all-workspaces` and web Show-all listing — operator-scope: requests carrying agent identity receive the standard `403 agent_scope_denied`.
- `GET/PATCH /api/settings/attention` — settings section mirroring `[attention]`.

## config.toml

```toml
[attention]
# In-app toasts for needs-you transitions and coalesced completions.
toasts = true
# Single built-in sound, follows the same suppression rules as its notification.
sound = true
# System-level (OS) notifications; fire only while the app is unfocused/hidden.
system = false
# Workspace IDs whose notifications are silenced (bell rows and counts remain).
muted_workspaces = []
```

- All four keys apply live (no daemon restart).
- `[window_manager.shortcuts]` values widen: each action accepts a chord string **or** an array of chords; indexed families (`desktop.switch`, `window.tab.jump`) accept one `"…Digit1..9"` range string. Old single-string form remains the one-element case of the same grammar.

```toml
[window_manager.shortcuts]
"window.focus.left" = ["control+ArrowLeft", "alt+KeyH"]
"desktop.switch" = "meta+control+Digit1..9"
"session.focus.attention" = "alt+KeyI"
```

## Keyboard Defaults

The frozen default keymap (new, revised, and migrated actions; unchanged actions keep today's chords). `meta` renders ⌘ on Apple and maps to Ctrl elsewhere for single-primary chords.

| Action | Default | Note |
| --- | --- | --- |
| `palette.open` | ⌘K, ⌘⇧P | migrated to registry; ⌘⇧P = editor-palette affinity; both fire inside inputs |
| `palette.view.sessions` | ⌘E | opens the palette pushed into the Sessions view |
| `session.new` | ⌘N | migrated to registry; fires inside inputs |
| `scope.global.toggle` | ⇧⌘G | migrated to registry |
| `window.nav.back` | ⌘[ | migrated to registry (now rebindable and listed) |
| `session.cycle.next` / `.previous` | ⌃⇧↓ / ⌃⇧↑ | cycles the sidebar's visible order |
| `session.focus.attention` | ⌃⌥A | jump to the most recent attention session (cross-workspace) |
| `workspace.picker` | ⌘⇧O | opens the workspaces overview |
| `workspace.cycle.next` / `.previous` | ⌘⇧] / ⌘⇧[ | follows the menubar's visible workspace order |
| `desktop.switch.1..9` | ⌃1..⌃9 | range family; macOS Spaces affinity |
| `desktop.create` | ⌘⇧N | — |
| `desktop.overview` | ⌘⇧D | revised from ⇧⌘S |
| `desktop.switch.previous` / `.next` | ⌃⇧← / ⌃⇧→ | revised from ⌃⌥[ / ⌃⌥] |
| `sidebar.toggle` | ⌘B | editor affinity |
| `window.focus.last` | ⌃⌥O | most-recently-used window toggle |
| `window.zoom` | ⌃⌥Z | revised from ⌃⌥↑ |
| `window.tile.top` / `.bottom` | ⌃⌥↑ / ⌃⌥↓ | newly bound (were unbound); tiles now cover ←→↑↓ + 4 corners |
| `shortcuts.cheatsheet` | ?, ⌘/ | ⌘/ is the reliable chord on non-US layouts |

Precedence: focused UI wins over global chords (the palette's own list navigation, editor fields, docks). Known platform notes documented on the shortcuts docs page: ⌃digits collide with browser tab switching outside the desktop app on non-Apple platforms; ⌃⌥ chords alias AltGr on some layouts.

### Terminal preset (applied from Settings → Shortcuts; pasteable equivalent)

```toml
[window_manager.shortcuts]
"window.focus.left" = ["control+ArrowLeft", "alt+KeyH"]
"window.focus.down" = ["control+ArrowDown", "alt+KeyJ"]
"window.focus.up" = ["control+ArrowUp", "alt+KeyK"]
"window.focus.right" = ["control+ArrowRight", "alt+KeyL"]
"window.focus.last" = ["control+alt+KeyO", "alt+KeyO"]
"window.close" = ["meta+KeyW", "alt+KeyW"]
"window.zoom" = ["control+alt+KeyZ", "alt+KeyY"]
"window.tab.new" = ["meta+KeyT", "alt+KeyT"]
"window.tab.next" = ["control+Tab", "control+alt+KeyL"]
"window.tab.previous" = ["control+shift+Tab", "control+alt+KeyH"]
"window.tab.jump" = "control+alt+Digit1..8"
"window.tab.last" = "control+alt+Digit9"
"session.cycle.next" = ["control+shift+ArrowDown", "control+alt+KeyJ"]
"session.cycle.previous" = ["control+shift+ArrowUp", "control+alt+KeyK"]
# the two bottom tiles move off control+alt+J/K to make room for session cycling:
"window.tile.bottom-left" = "control+alt+shift+KeyJ"
"window.tile.bottom-right" = "control+alt+shift+KeyK"
"session.focus.attention" = ["control+alt+KeyA", "alt+KeyI"]
"workspace.picker" = ["meta+shift+KeyO", "meta+control+KeyP"]
"workspace.cycle.next" = ["meta+shift+BracketRight", "meta+control+KeyJ"]
"workspace.cycle.previous" = ["meta+shift+BracketLeft", "meta+control+KeyK"]
"desktop.create" = ["meta+shift+KeyN", "meta+control+KeyN"]
"desktop.switch" = ["control+Digit1..9", "meta+Digit1..9"]
```

The preset frees ⌘1..9 for desktops by moving tab jumps to ⌃⌥ digits, and moves the two bottom tiles to the ⌃⌥⇧ layer to free ⌃⌥J/K for session cycling — the preview shows exactly that diff (every displaced default included) before applying, the block passes full effective-map validation as-is, and revert restores the pre-preset state in one step.

## Native Tools

### `compozy__session_wait`

```json
// call
{ "session_id": "ses_01jm2e6r9k", "until": ["waiting-for-input", "idle"], "timeout_ms": 120000 }

// result
{ "session_id": "ses_01jm2e6r9k", "outcome": "state-reached", "state": "waiting-for-input", "waited_ms": 61000 }
```

Read-only; same-workspace; `timeout_ms` default `300000`, hard cap `1800000`; waiting on self → denied. Outcomes: `state-reached`, `timeout`, `session-gone`, `canceled`.

### `compozy__session_spawn`

```json
// call
{ "agent_name": "researcher", "ttl_seconds": 3600, "prompt_overlay": "Audit our HTTP client wrappers", "notify_creator": true }

// result
{ "session_id": "ses_01jm2e6r9k", "spawn_role": "worker", "spawn_depth": 1, "ttl_expires_at": "2026-08-15T18:30:00Z" }
```

Same governance as `POST /api/agent/spawn` (TTL required, depth/children caps, permission subset via `tools`/`skills`/`mcp_servers`/`workspace_paths`/`network_channels`/`sandbox_profiles` arrays).

### `compozy__session_stop`

```json
{ "session_id": "ses_01jm2e6r9k" }
→ { "session_id": "ses_01jm2e6r9k", "state": "stopped" }
// already stopped → { "state": "stopped", "outcome": "already-stopped" }
```

Destructive risk class (approval policy applies). Stopping self → denied.

### `compozy__session_approve`

```json
{ "session_id": "ses_01jm1zqv04", "request_id": "perm_3d8a", "decision": "allow-once" }
→ { "outcome": "applied", "request_id": "perm_3d8a", "decision": "allow-once" }
// raced → { "outcome": "already-resolved", "resolved_decision": "reject-once" }
```

Decisions: `allow-once`, `allow-always`, `reject-once`, `reject-always`. Approving own request → denied.

### `compozy__session_clarify_answer`

```json
{ "session_id": "ses_01jm2e6r9k", "request_id": "clr_7f2a", "choice": 2 }
→ { "outcome": "answered", "request_id": "clr_7f2a" }
```

`choice` (1-based) or `text`, exactly one. Answering own question → denied.

### `compozy__notify`

```json
{ "title": "Deps audit done", "body": "3 findings, 1 high severity" }
→ { "outcome": "delivered" }
```

Same outcomes and limits as `compozy notify` / `POST /api/agent/notify`.

### `compozy__session_prompt_cancel`

```json
{ "session_id": "ses_01jm2e6r9k" }
→ { "session_id": "ses_01jm2e6r9k", "outcome": "canceled", "turn_id": "tur_8c1d" }
// idle target → { "session_id": "ses_01jm2e6r9k", "outcome": "nothing-in-flight" }
```

Mutating risk class; same-workspace; the session survives — only the in-flight turn ends, through the same cancellation path as the CLI verb and route.

## Hooks (extension surface)

New typed event, observable by extensions and config-registered hooks:

```
session.attention.changed
{ "session_id": "ses_01jm2e6r9k", "workspace_id": "ws_api",
  "from": "running", "to": "waiting-for-input", "class": "needs-you",
  "at": "2026-08-15T18:04:13.512Z" }
```

`class` ∈ `needs-you` · `finished` · `none`; `at` is the durable transition timestamp (the same ordering key surfaces expose). Dispatch semantics: post-commit (after the durable state change), asynchronous, payload cloned, observational and fail-open — a hook error is logged and never blocks or mutates the transition.

## Errors

| Condition | Surface | What the caller sees |
| --- | --- | --- |
| Unknown state token in `--until`/`until` | CLI / API / tool | exit `65` / `422 invalid_state` / `invalid_input` naming the valid vocabulary |
| Wait timeout | CLI / API / tool | exit `75` / `200 {"outcome":"timeout"}` / `{"outcome":"timeout"}` — always distinguishable from errors |
| Session deleted mid-wait | CLI / API / tool | exit `69` "session gone" / `410 session_gone` / `{"outcome":"session-gone"}` |
| Wait/stop/approve/answer on self | tool | `denied` with reason `approval_self_denied` (approve/answer) or `self_target_denied` (wait/stop) |
| Cross-workspace target | tool | standard workspace denial (same as every session tool) |
| Approve/answer already resolved | CLI / API / tool | `outcome: already-resolved` + winning decision — never an error |
| Notify rate limit | CLI / API / tool | `outcome: rate-limited`, `retry_after_ms` — never an error |
| Notify title/body over cap | CLI / API / tool | exit `65` / `422 invalid_notification` / `invalid_input` naming the limit |
| Spawn caps exceeded | API / tool | `409 spawn_limit_exceeded` naming cap and usage |
| Spawn permission widening | API / tool | `422 spawn_validation` listing offending atoms |
| Prompt-cancel with nothing in flight | CLI / API / tool | exit `66` / `200 {"outcome":"nothing-in-flight"}` / `{"outcome":"nothing-in-flight"}` |
| Agent identity on an operator surface (presence, cross-workspace listing, attention summary) | API / CLI | `403 agent_scope_denied` — same shape on both transports |
| Orphaned-interaction resolution with a full input queue | CLI / API / tool | retryable `outcome: queue-full` — the pending record is untouched, retry later |
| Resuming a wait after the 10s grace expired | API / CLI | `wait-expired` — re-register a fresh wait (the CLI does this transparently) |
| Wait edge-buffer overflow (64 edges) | API / CLI / tool | `outcome: overflow` — deterministic, re-register; never silent loss |
| Duplicate chord (incl. array members and expanded ranges) | CLI / settings | exit `78` / `422` naming both actions and the chord |
| Range on non-indexed action | CLI / settings | exit `78` / `422` naming the range-capable actions |
| Muted workspace notify | tool / CLI | `outcome: muted-workspace` — truthful, never silent success |
