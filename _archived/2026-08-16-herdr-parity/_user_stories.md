# User Stories: Herdr Parity (Session Attention · Orchestration DX · Shortcuts v2)

Canonical behavior catalog for the herdr-parity program. Companion to `_spec.md`; consumed by
`_spec.md` Part II (component mapping), `_uiux.md` (surface states), and
`_tests.md` (coverage matrix).

## Personas

- **Operator** — a human running many concurrent agent sessions in the CompozyOS shell (web or desktop app), across multiple workspaces, desktops, and worktrees. Needs to know instantly which session needs them and to move between sessions without touching the mouse.
- **Orchestrator agent** — a Compozy session (usually a parent) that manages sibling/child sessions programmatically through structured surfaces: it spawns workers, waits for them to settle, unblocks them, and reports to the operator.
- **Headless operator** — a human (or script) working from a terminal with no web UI open: inspects, filters, waits on, and unblocks sessions using CLI commands with machine-readable output.

## Story Index

| ID     | Feature Area          | Persona            | Story                                                        |
| ------ | --------------------- | ------------------ | ------------------------------------------------------------ |
| US-001 | Attention states      | Operator           | Distinguish "asked a question" from "needs permission"        |
| US-002 | Attention states      | Operator           | See which sessions finished while I was not looking (`done`)  |
| US-003 | Attention states      | Operator           | Failed sessions demand my attention                           |
| US-004 | Attention states      | Headless operator  | Inspect and filter attention states from the CLI              |
| US-005 | Attention surfaces    | Operator           | Bell groups "Needs you" and "Finished" and jumps to a session |
| US-006 | Attention surfaces    | Operator           | Tab/app title shows the awaiting count                        |
| US-007 | Attention surfaces    | Operator           | Sort the sessions sidebar attention-first                     |
| US-008 | Notifications         | Operator           | Get notified when a session needs me                          |
| US-009 | Notifications         | Operator           | Completion notifications arrive coalesced                     |
| US-010 | Notifications         | Operator           | Hear a sound when attention is required                       |
| US-011 | Notifications         | Operator           | Configure notification behavior everywhere I work             |
| US-012 | Notifications         | Operator           | Escalate to system-level notifications when app unfocused     |
| US-013 | Notifications         | Orchestrator agent | Send the operator a notification                              |
| US-014 | Cross-workspace       | Operator           | See and reach attention across all workspaces                 |
| US-015 | Cross-workspace       | Operator           | Mute a noisy workspace without losing its counters            |
| US-016 | Orchestration: wait   | Headless operator  | Wait until a session reaches a state, from the CLI            |
| US-017 | Orchestration: wait   | Orchestrator agent | Wait natively without a shell                                 |
| US-018 | Orchestration: tools  | Orchestrator agent | Spawn a governed child session natively                       |
| US-019 | Orchestration: tools  | Orchestrator agent | Be woken when my child settles or blocks                      |
| US-020 | Orchestration: tools  | Orchestrator agent | Stop a session natively                                       |
| US-021 | Orchestration: tools  | Orchestrator agent | Approve or deny a session's permission request natively       |
| US-022 | Orchestration: tools  | Orchestrator agent | Answer a session's question natively                          |
| US-023 | Orchestration: tools  | Headless operator  | Cancel an in-flight prompt from the CLI                       |
| US-024 | Shortcuts grammar     | Operator           | Bind several chords to one action                             |
| US-025 | Shortcuts grammar     | Operator           | Bind numbered ranges in one line                              |
| US-026 | Shortcuts grammar     | Operator           | Apply the terminal preset in one click                        |
| US-027 | Shortcut actions      | Operator           | Navigate sessions, desktops, and attention by keyboard        |
| US-028 | Shortcut actions      | Operator           | Trust the cheatsheet and rebind everything                    |
| US-029 | Palette views         | Operator           | Navigate nested palette views                                 |
| US-030 | Palette views         | Operator           | Switch sessions through the palette Sessions view             |
| US-031 | Cross-workspace       | Operator           | "Show all" reveals every workspace's sessions in any list     |

---

## Attention states

### US-001: Distinguish "asked a question" from "needs permission"

**As an** operator, **I want** a session that asked me a question to show a distinct needs-input state (separate from the permission-wait state), **so that** I know whether I must compose an answer or just approve/deny — before opening the session.

Acceptance criteria:

- AC-1: Given a session whose agent asked a clarifying question, when I look at the sessions sidebar, then the session shows the needs-input state (not "running") within the live-refresh latency target.
- AC-2: Given a session waiting on a permission request, when I look at any session surface, then it shows the needs-auth state, visually in the same "needs you" class as needs-input but textually distinct.
- AC-3: Given a session in needs-input, when I answer the question (from any surface), then the state leaves the "needs you" class everywhere without manual refresh.
- AC-4: State is never conveyed by color alone: every state indicator carries a distinct shape or glyph and an accessible text label.

Edge cases:

- EC-1: Session has a pending question AND a pending permission simultaneously → one state wins by a defined precedence (needs-auth outranks needs-input); answering the winner reveals the loser, never a flicker of "idle" in between.
- EC-2: Daemon restarts while a question/permission is pending → the state survives the restart; the session does not silently show running/idle.
- EC-3: The question is answered by another client or an orchestrator agent → all surfaces clear within the live-refresh target; no stale "needs you" remains.
- EC-4: The agent withdraws or times out its own question → the state clears without operator action and the session returns to its underlying state.
- EC-5: A session that never reports activity (unsupported/foreign agent) → shows the unknown state honestly, never a confidently wrong needs-you or idle.

### US-002: See which sessions finished while I was not looking (`done`)

**As an** operator, **I want** sessions that finished a turn while I was not viewing them to carry a distinct finished-unseen (`done`) marker until I look, **so that** completed work forms an inbox I can drain instead of a state I must memorize.

Acceptance criteria:

- AC-1: Given a session finishes its turn while its window is not focused (or the app is hidden), when I next look at the sidebar/bell, then the session shows `done` (distinct finished-unseen tone and glyph) instead of plain idle.
- AC-2: Given a `done` session, when I focus its window with the app visible, then the marker clears to idle automatically — no manual dismiss exists anywhere.
- AC-3: Given a session finishes while I am actively viewing it, then it never shows `done` — it goes straight to idle.
- AC-4: Only the system derives `done`; no agent, tool, or client can set or report it.

Edge cases:

- EC-1: CLI reads (status, list) and agent read tools touch the session → `done` does NOT clear; only viewing the session in the shell clears it.
- EC-2: Daemon restarts after a session went `done` → the marker survives; already-seen sessions do not resurrect as `done`.
- EC-3: Two clients have the app open; one focuses the session → `done` clears for both within the live-refresh target.
- EC-4: The session starts a new turn while still unseen → `done` yields to running; if the new turn ends unseen again, `done` returns.
- EC-5: Session goes `done`, then is archived or stopped before being seen → terminal/archived presentation wins; the unseen marker does not outrank lifecycle states.
- EC-6: 100+ sessions flip to `done` in bulk (batch orchestration) → surfaces stay responsive and the "Finished" section paginates or scrolls without truncating silently.

### US-003: Failed sessions demand my attention

**As an** operator, **I want** failed sessions to participate in the needs-you attention class, **so that** a crashed worker is as loud as a blocked one. (`unhealthy`/`hung` stay warning-grade states outside needs-you.)

Acceptance criteria:

- AC-1: Given a session transitions to failed, then it appears in the bell "Needs you" section, counts in the needs-you counter, and fires a notification like any needs-you transition.
- AC-2: Given the failure is resolved (session repaired, resumed, or removed), then it leaves the counters and bell without manual dismissal.

Edge cases:

- EC-1: A session fails while its window is focused → no notification (focus suppression), but counter/bell still update.
- EC-2: A session flaps failed→running→failed rapidly → the per-session dedup window prevents a notification burst; the state shown is always the current one.
- EC-3: Session fails in a muted workspace → row and counter appear; no toast/sound/system alert.

### US-004: Inspect and filter attention states from the CLI

**As a** headless operator, **I want** session listings and status output to carry the full attention vocabulary and support filtering to "sessions that need me", **so that** scripts and terminal workflows see exactly what the shell sees.

Acceptance criteria:

- AC-1: Given sessions in needs-input/needs-auth/done/failed states, when I list sessions with the attention filter, then exactly those in the needs-you class return, with the state named per session in both human and machine-readable output.
- AC-2: Given a specific session, when I query its status, then the same state the web shell renders is reported — one vocabulary everywhere (CLI, web, agent tools).
- AC-3: The `done` marker is visible in listings (read-only) and querying it never clears it.

Edge cases:

- EC-1: No sessions match the filter → empty result with success exit semantics (not an error).
- EC-2: Filter by a state name that does not exist → deterministic validation error naming the allowed vocabulary.
- EC-3: Machine-readable output is stable across releases: state tokens are part of the public contract, renames are hard cuts (no aliases).

---

## Attention surfaces

### US-005: Bell groups "Needs you" and "Finished" and jumps to a session

**As an** operator, **I want** the menubar bell to show needs-you rows and finished-unseen rows in separate sections, and clicking a row to take me to that session, **so that** the bell is both an alarm and an inbox.

Acceptance criteria:

- AC-1: Given sessions needing me and sessions finished-unseen, when I open the bell, then "Needs you" (questions, permissions, failures) and "Finished" render as distinct sections, needs-you first.
- AC-2: The bell badge number equals the needs-you count only; finished-unseen never inflates it.
- AC-3: Given I activate a row, then the session's window is focused (workspace switched first if the row belongs to another workspace) and, for `done` rows, the seen-marker clears on arrival.
- AC-4: Pending task approvals keep their existing rows alongside session attention.

Edge cases:

- EC-1: Zero attention anywhere → bell renders its quiet state; no empty sections.
- EC-2: The row's session changes state (or is stopped/archived/deleted) between render and click → the jump lands gracefully: focus if it still exists, otherwise an honest "no longer available" outcome; never a broken window.
- EC-3: 100+ rows → sections stay scrollable and ordered (most recent transition first); counts remain exact, not "99+" unless the design says so explicitly.
- EC-4: Two clients open the bell; one acts on a row → the other's list updates within the live-refresh target.

### US-006: Tab/app title shows the awaiting count

**As an** operator, **I want** the browser tab (and desktop app) title to carry the needs-you count, **so that** a backgrounded CompozyOS still signals me with zero setup and zero permission prompts.

Acceptance criteria:

- AC-1: Given N sessions in the needs-you class (any workspace), when the app is backgrounded, then the title shows the count; when N changes, the title updates without the tab being focused.
- AC-2: Given N returns to zero, then the title returns to its clean form.

Edge cases:

- EC-1: Counts differ per workspace → the title uses the cross-workspace total (same number as the bell).
- EC-2: Title updates never fight the page's own title changes (navigation) — the count survives route changes.

### US-007: Sort the sessions sidebar attention-first

**As an** operator, **I want** an attention-first sort option for the sessions sidebar, **so that** sessions needing me float to the top, most recent transition first.

Acceptance criteria:

- AC-1: Given the attention-first sort is selected, then ordering is: needs-you (newest transition first), then finished-unseen, then working, then idle/rest.
- AC-2: The chosen sort persists across app restarts for that operator.

Edge cases:

- EC-1: Ties within a class → stable secondary order (most recent activity), no row jitter on refresh.
- EC-2: A session transitions while the list is visible → it animates/moves to its new position without losing selection focus for keyboard users.

---

## Notifications

Vocabulary note: "notification" in this catalog means the user-facing attention alert (in-app toast, system-level alert, sound, title count). It is distinct from the existing bridge notification presets (task fan-out to external messengers), which are untouched by this program.

### US-008: Get notified when a session needs me

**As an** operator, **I want** an immediate notification when any session transitions into needs-input, needs-auth, or failed, **so that** blocked agents never wait on me silently.

Acceptance criteria:

- AC-1: Given a session transitions into a needs-you state, when its window is not focused (or the app is hidden), then an in-app toast appears naming the session and the reason (question / permission / failure).
- AC-2: Given the session's window is focused and the app visible, then no notification fires for that transition (focus suppression) — surfaces still update.
- AC-3: Given I activate the toast, then the session's window is focused (switching workspace if needed) and the toast dismisses.
- AC-4: Needs-you notifications are per-session and immediate; a per-session dedup window (a few seconds) absorbs rapid flapping of the same session.

Edge cases:

- EC-1: Five sessions hit needs-you within a second → five distinct toasts (each needs a distinct action); the stack remains readable and dismissible.
- EC-2: The same session flaps needs-you → running → needs-you inside the dedup window → one notification, current state shown.
- EC-3: A transition happens while the daemon restarts → after restart the surviving pending state is reflected in counters/bell; no duplicate "new" notification storm for states already notified.
- EC-4: Toast arrives for a session that resolves before I click → clicking lands on the session in its current state; no error.

### US-009: Completion notifications arrive coalesced

**As an** operator, **I want** "finished" notifications grouped when several sessions complete near-simultaneously, **so that** a fleet finishing does not bury me in alerts.

Acceptance criteria:

- AC-1: Given one unfocused session finishes, then one toast reports it finished.
- AC-2: Given multiple sessions finish within the coalescing window, then one grouped toast reports "N sessions finished" and activating it opens the bell's Finished section.
- AC-3: Completion notifications never fire for a session finishing in the focused window.

Edge cases:

- EC-1: A completion and a needs-you land together → needs-you notifies individually; the completion still coalesces separately — the urgent signal is never grouped away.
- EC-2: Completions trickle just outside the window → separate toasts; the window never extends itself indefinitely (no starvation of the signal).
- EC-3: A session finishes and is immediately seen (I focus it) before the coalesced toast fires → it drops out of the group; a group of one collapses to the single form, a group of zero fires nothing.

### US-010: Hear a sound when attention is required

**As an** operator, **I want** an audible cue with notifications, on by default with one global toggle, **so that** I catch attention moments while looking away from the screen.

Acceptance criteria:

- AC-1: Given sound is enabled (default), when a notification fires, then the built-in sound plays once per notification event (a coalesced group plays once).
- AC-2: Given sound is disabled, notifications stay visual-only everywhere.
- AC-3: Sound obeys the same suppression rules as its notification (focus suppression, workspace mute).

Edge cases:

- EC-1: Many notifications in one instant → sounds do not overlap into a chorus; at most one sound per delivery batch.
- EC-2: The client environment cannot play audio (browser autoplay policy, no output device) → visual delivery is unaffected and no error surfaces to the operator.

### US-011: Configure notification behavior everywhere I work

**As an** operator, **I want** notification settings (toast on/off, system-level on/off, sound on/off, per-workspace mute) manageable from the settings UI, the configuration file, and the CLI with structured output, **so that** I and my agents can inspect and repair the setup without the web UI.

Acceptance criteria:

- AC-1: Given I change any notification setting in the settings UI, then behavior changes take effect without restarting the daemon, and the configuration file reflects the change.
- AC-2: Given I read or set the same keys via the CLI, then values round-trip identically (UI, file, CLI agree), with machine-readable get/list output.
- AC-3: Given an invalid value, then a deterministic validation error names the key and the allowed values — the setting never half-applies.

Edge cases:

- EC-1: First run (no keys present) → documented defaults apply: toast on, sound on, system-level off, no workspaces muted.
- EC-2: Concurrent writers (two settings surfaces at once) → last write wins consistently; no torn state between delivery channels.
- EC-3: A muted workspace is deleted → its mute entry is cleaned up; listing settings never shows orphans.

### US-012: Escalate to system-level notifications when app unfocused

**As an** operator, **I want** opt-in system-level (OS) notifications that fire only while the app is unfocused or hidden, **so that** attention reaches me outside CompozyOS without ever double-firing on top of a toast I can already see.

Acceptance criteria:

- AC-1: Given system-level notifications are enabled and the platform permission is granted, when a notification-worthy transition happens while the app is unfocused/hidden, then a system notification appears; activating it focuses the app and the session (workspace switch included).
- AC-2: Given the app is focused and visible, system-level notifications never fire — the toast (or focus suppression) owns that case.
- AC-3: The setting is off by default; enabling it walks the platform permission flow honestly.

Edge cases:

- EC-1: Platform permission denied or revoked → the setting surface shows the channel as unavailable-with-reason; in-app delivery continues; no silent pretend-armed state.
- EC-2: Browser vs desktop app differences in capability → each states its actual capability; no control renders for an unsupported channel (truthful UI).
- EC-3: System notification clicked after the session resolved → app focuses and lands on the session's current state without error.

### US-013: Send the operator a notification

**As an** orchestrator agent, **I want** a native way to send the operator a short notification (title/body), **so that** long-running work can announce milestones or ask for eyes without abusing question/permission states.

Acceptance criteria:

- AC-1: Given an agent invokes the notify surface with a title and optional body, then the operator receives it through the same delivery ladder (toast, optional system-level, sound, bell presence per design) with the sending session identified.
- AC-2: Content is sanitized and length-capped; the result reports a daemon-provable outcome deterministically (delivered / no-client / muted-workspace / rate-limited) — the system reports what it can prove (delivery to a live client), never whether a human saw it.
- AC-3: A per-session rate limit bounds notification frequency; exceeding it returns the rate-limited outcome, never an exception.

Edge cases:

- EC-1: Title empty or over the cap → deterministic validation error naming limits; nothing partial is shown.
- EC-2: Agent spams the surface → rate limit engages; counters/log make the suppression observable; the operator's channel never floods.
- EC-3: Notification sent from a session in a muted workspace → muted outcome reported to the agent; no silent success lie.
- EC-4: Sent while the operator has no client connected → outcome `no-client` — the agent can tell nothing was delivered live.

---

## Cross-workspace attention

### US-014: See and reach attention across all workspaces

**As an** operator, **I want** the bell, counters, and notifications to cover every workspace — not just the active one — and activating any of them to carry me to the right workspace and session, **so that** no workspace can stall silently in the background.

Acceptance criteria:

- AC-1: Given sessions need me in workspaces A and B while C is active, then the bell badge counts them all, rows are labeled with their workspace, and notifications fire for A and B.
- AC-2: Given I activate a row/notification for workspace A, then the shell switches to A and focuses that session's window in one motion.
- AC-3: The awaiting count in the title equals the cross-workspace needs-you total.

Edge cases:

- EC-1: The target workspace was removed after the row rendered → honest failure ("workspace no longer exists"), row disappears on next refresh.
- EC-2: Jump into a workspace whose windows/desktops don't include that session window yet → the shell opens/focuses the session window rather than landing nowhere.
- EC-3: Many workspaces with attention at once → rows group or label by workspace so the list stays scannable.
- EC-4: A workspace the operator cannot access anymore (permission changed) → its rows vanish rather than erroring on click.

### US-015: Mute a noisy workspace without losing its counters

**As an** operator, **I want** a per-workspace mute that silences a workspace's notifications (toast/system/sound) while keeping its bell rows and counts, **so that** a chatty batch workspace stops interrupting me but never becomes invisible.

Acceptance criteria:

- AC-1: Given workspace A is muted, when its sessions transition into needs-you or done, then no toast/system alert/sound fires, but the bell rows, badge count, and title count still include A.
- AC-2: Mute state is settable from the settings UI, configuration file, and CLI, and is visible wherever notifications are configured.

Edge cases:

- EC-1: Muting while a coalescing window is open → pending grouped completions for that workspace are dropped from delivery, not from the bell.
- EC-2: Unmuting → only new transitions notify; no replay burst of everything missed while muted.

### US-031: "Show all" reveals every workspace's sessions in any session list

**As an** operator, **I want** a "Show all" option on every session list surface — the sessions sidebar, the palette Sessions view, and the operator CLI listing — **so that** I can see and reach every session across all workspaces from wherever I already am.

Acceptance criteria:

- AC-1: Given Show all is enabled in the sidebar, then sessions from every workspace render grouped and labeled by workspace, with the same state indicators and sort semantics; activating a row from another workspace performs the standard cross-workspace jump (switch, then focus).
- AC-2: Given the palette Sessions view is open, then the same option widens the list from the active workspace (default) to every workspace; state filters and attention-first ordering apply across the widened set.
- AC-3: Given the operator CLI session listing is invoked with the all-workspaces option, then sessions list with their workspace named, in human and machine-readable output — an operator affordance only; agent-facing session tools remain same-workspace scoped.
- AC-4: The Show all choice defaults to off and persists per surface for the operator across restarts.

Edge cases:

- EC-1: One workspace fails to load while Show all is on → the other workspaces still render; the failing one shows an honest per-group error instead of sinking the list.
- EC-2: Hundreds of sessions across many workspaces → grouped list stays responsive (scroll/virtualize); attention-first ordering still floats needs-you rows to the top of each group.
- EC-3: A workspace is removed while shown → its group disappears on the next refresh; activating a stale row degrades honestly (US-014.EC-1 parity).
- EC-4: A new workspace appears while Show all is on → its sessions join the list within the live-refresh target, without reload.

---

## Orchestration: wait

### US-016: Wait until a session reaches a state, from the CLI

**As a** headless operator, **I want** the session wait command to accept a target state set and a timeout — not just "until stopped" — **so that** scripts block precisely on "worker needs input", "worker settled", or "worker finished" instead of poll-looping.

Acceptance criteria:

- AC-1: Given a running session, when I wait with an explicit state set (e.g. needs-input, needs-auth, idle, stopped, failed), then the command returns as soon as the session enters any of them, reporting which state fired and when, in human and machine-readable form.
- AC-2: Given no explicit set, then a documented settled-set default applies (the session "settled or needs someone": needs-input, needs-auth, idle, stopped, failed) — `done` satisfies idle.
- AC-3: Given a timeout is set and elapses first, then the command exits with a distinct, documented timeout exit code and a deterministic message — distinguishable from "state reached" and from transport errors.
- AC-4: Given the session is already in a target state when the wait starts, then it returns immediately with that state (no lost-edge hang).
- AC-5: The wait is pinned to the session identity it started with: a session deleted and replaced under the same name/route never satisfies the original wait.

Edge cases:

- EC-1: Session deleted/archived mid-wait → deterministic "session gone" outcome, not a hang or a false success.
- EC-2: Daemon connection drops mid-wait → the command reconnects or fails deterministically per design; it never silently misses the edge and hangs forever.
- EC-3: Invalid state token in the set → validation error naming the allowed vocabulary; nothing starts.
- EC-4: Two concurrent waits on the same session → both fire independently on the same transition.
- EC-5: Wait for a state the session is transitioning through rapidly (enter/leave within the refresh interval) → the wait still fires (edge-driven, not sampled).
- EC-6: Timeout of zero / negative → validation error; unbounded waits require an explicit opt-in flag, never a silent default-forever.
- EC-7: Unbounded wait spanning many internal continuations → a state entered and left between continuations still satisfies the wait (gapless, edge-driven end to end).

### US-017: Wait natively without a shell

**As an** orchestrator agent, **I want** a native wait tool with the same semantics as the CLI wait, **so that** I can turn "poll status in a loop" into one event-driven call even when I have no shell access.

Acceptance criteria:

- AC-1: Given a target session in my workspace, when I invoke the native wait with a state set and timeout, then the result reports the terminal outcome (state-reached with the state and timestamp, timeout, or session-gone) as structured data.
- AC-2: Same-workspace scoping and existing session-tool permissions apply; waiting is read-only (never mutates the target).
- AC-3: The tool's timeout is bounded by a documented cap so a stuck wait cannot pin an agent turn forever.

Edge cases:

- EC-1: Target session outside my workspace → the same deterministic denial every session tool returns.
- EC-2: Wait invoked on self → validation error (an agent cannot block on its own settling).
- EC-3: Tool call canceled/interrupted mid-wait (turn canceled) → the wait aborts cleanly with a canceled outcome; no orphaned watcher remains.
- EC-4: Timeout requested above the cap → clamped or rejected deterministically per design, documented in the tool schema.

---

## Orchestration: lifecycle tools

### US-018: Spawn a governed child session natively

**As an** orchestrator agent, **I want** to spawn a child session through a native tool carrying the same governance as the existing spawn surfaces (TTL, depth cap, children cap, permission narrowing), **so that** delegation does not require shell access.

Acceptance criteria:

- AC-1: Given valid spawn parameters, then a child session starts, the result returns its identity, and lineage (parent/root/depth) is recorded exactly as with the operator spawn path.
- AC-2: All governance rules apply unchanged: TTL mandatory, depth/children caps enforced, child permissions a strict subset (widening atoms reject).
- AC-3: The spawn accepts the wake-on-settle choice (US-019) with a documented default of on.

Edge cases:

- EC-1: Caps exceeded (depth or children) → deterministic error naming the cap and current usage.
- EC-2: Requested permission atoms widen the parent's → rejected with the offending atoms listed.
- EC-3: Spawn during parent shutdown → rejected cleanly; no orphan child.
- EC-4: Duplicate rapid spawns (retry after unclear result) → each call is a distinct child; the result carries the identity so the agent can detect its own duplicates.

### US-019: Be woken when my child settles or blocks

**As an** orchestrator agent, **I want** to be woken (prompted) when a child session I spawned reaches a terminal state, fails, or enters a needs-you state, **so that** I resume orchestration event-driven instead of polling or waiting synchronously.

Acceptance criteria:

- AC-1: Given I spawned a child with wake-on-settle on (default), when the child stops, fails, or blocks on input/permission, then I receive a wake prompt naming the child, the reason, and enough identity to act (mirroring the existing task wake behavior).
- AC-2: Given wake-on-settle was disabled at spawn, no wake is delivered for that child.
- AC-3: Wakes are suppressed with an observable reason when delivery is impossible or nonsensical (parent stopped, self-wake, disabled) — suppression is auditable, never silent loss.

Edge cases:

- EC-1: Parent is mid-turn when the child settles → the wake queues per the existing wake delivery semantics; it is never dropped because the parent was busy.
- EC-2: Parent stopped before the child settles → suppression recorded with reason; no crash, no zombie delivery.
- EC-3: Child flaps into needs-you repeatedly → wakes for the same cause are deduplicated per cause-episode, not per event.
- EC-4: Child and parent settle simultaneously (parent stopping while child stops) → at most one coherent outcome; no deadlock between reaper and wake.

### US-020: Stop a session natively

**As an** orchestrator agent, **I want** to stop a sibling/child session through a native tool, **so that** cleanup and cancellation do not require shell access.

Acceptance criteria:

- AC-1: Given a stoppable session in my workspace, invoking stop ends it with the same semantics as the CLI stop, and the result confirms the final state.
- AC-2: Stopping an already-stopped session is idempotent (success, no-op indicated).
- AC-3: The tool is risk-gated like other destructive tools: subject to permission mode and approval policy.

Edge cases:

- EC-1: Stop own session → rejected with a deterministic error (self-stop is the provider's lifecycle, not a tool call).
- EC-2: Stop a session mid-prompt → in-flight work cancels per the one idempotent cancellation path; events record the actor.
- EC-3: Cross-workspace target → the standard workspace denial.

### US-021: Approve or deny a session's permission request natively

**As an** orchestrator agent, **I want** to answer a child's pending permission request (approve/deny) through a native tool, **so that** supervised automation can unblock workers without a human when policy allows.

Acceptance criteria:

- AC-1: Given a session with a pending permission request, invoking approve/deny resolves it with the same effect as the operator surfaces, recording the acting session as the decision actor.
- AC-2: A session can never approve its own permission request — deterministic rejection.
- AC-3: The result names the decision applied and the request it resolved.

Edge cases:

- EC-1: Request already answered (race with the operator) → deterministic already-resolved outcome, not an error cascade.
- EC-2: Request expired or session no longer waiting → honest "nothing pending" outcome.
- EC-3: Deny with a reason → the reason reaches the blocked session's transcript.
- EC-4: Resolving a request that survived a restart while the session's input queue is full → deterministic, retryable queue-full outcome; the pending request stays intact and resolvable (nothing is lost).

### US-022: Answer a session's question natively

**As an** orchestrator agent, **I want** to answer a child's pending clarifying question through a native tool, **so that** known answers flow back without a human or a shell.

Acceptance criteria:

- AC-1: Given a session with a pending question, invoking answer delivers it exactly as the operator answer path does, and the target session resumes.
- AC-2: A session can never answer its own question — deterministic rejection.
- AC-3: The needs-input state clears everywhere once answered (US-001.AC-3).

Edge cases:

- EC-1: Question already answered elsewhere → already-resolved outcome with the winning answer's identity.
- EC-2: Answer to a question id that never existed → deterministic not-found error.
- EC-3: Oversized answer → validated against the same limits as the operator path.

### US-023: Cancel an in-flight prompt from the CLI

**As a** headless operator, **I want** a CLI verb that cancels a session's in-flight prompt (parity with the existing programmatic cancel), **so that** runaway turns are stoppable from a terminal without killing the session.

Acceptance criteria:

- AC-1: Given a session with an active prompt, the cancel verb ends the in-flight turn (session survives, goes idle) and reports what was canceled.
- AC-2: Given no active prompt, the verb returns a deterministic "nothing in flight" outcome with success-adjacent exit semantics (documented).
- AC-3: Output is available in machine-readable form; exit codes distinguish canceled / nothing-to-cancel / error.

Edge cases:

- EC-1: Cancel twice (second lands after the first finished) → idempotent "nothing in flight".
- EC-2: Cancel racing natural turn completion → either outcome is coherent; no session left in a phantom-running state.
- EC-3: Cancel during provider shutdown → deterministic error; the one cancellation path never forks.

---

## Shortcuts grammar

### US-024: Bind several chords to one action

**As an** operator, **I want** an action to accept one chord or an array of chords, **so that** I can keep the default and add my muscle-memory binding side by side.

Acceptance criteria:

- AC-1: Given an action bound to two chords, both trigger it everywhere (shell, settings display, cheatsheet list them all).
- AC-2: Given a chord already used by another action (including inside arrays), validation rejects the write naming both actions and the colliding chord.
- AC-3: An empty binding disables the action's shortcut without removing the action.

Edge cases:

- EC-1: Duplicate chord within the same action's array → normalized or rejected deterministically (documented), never double-fires.
- EC-2: Array with one invalid chord among valid ones → whole write rejects atomically with the bad entry named.
- EC-3: Existing single-chord configurations continue to parse — the widened shape accepts the scalar form as a one-element array (single grammar, no dual code paths).

### US-025: Bind numbered ranges in one line

**As an** operator, **I want** range chords ("modifiers+1..9") for indexed actions (tab jump, desktop switch), **so that** nine bindings do not need nine lines.

Acceptance criteria:

- AC-1: Given a range binding on an indexed action family, each digit triggers the matching index, and the settings/cheatsheet render the family compactly.
- AC-2: A single-index override on top of a range wins for that index; validation reports range-vs-single collisions across the expansion.

Edge cases:

- EC-1: Range on a non-indexed action → validation error naming which actions accept ranges.
- EC-2: Partial ranges (1..4) → allowed and rendered honestly.
- EC-3: Range expansion colliding with an existing binding on one digit only → the diagnostic names the exact digit, not just "range conflicts".

### US-026: Apply the terminal preset in one click

**As an** operator, **I want** a built-in "terminal preset" (the Herdr-habit layer: direct alt navigation, workspace/desktop layers, jump-to-attention) applicable and reversible in one click from Settings, **so that** multiplexer muscle memory works in CompozyOS without hand-authoring twenty bindings.

Acceptance criteria:

- AC-1: Given I apply the preset, the affected actions rebind in one atomic step, the settings table reflects every change immediately, and a revert control restores prior bindings in one step.
- AC-2: Before applying, I see exactly which bindings will change (old → new), including collisions with my custom bindings, and the preset wins only after I confirm.
- AC-3: The preset is data, not a mode: after applying, individual bindings remain freely editable.

Edge cases:

- EC-1: Preset applied twice → idempotent; revert still returns to the pre-preset state, not to an intermediate.
- EC-2: Preset chord colliding with a custom binding on an action outside the preset → the preview names it; confirming reassigns deterministically.
- EC-3: Applying on a platform where a preset chord is hazardous (known layout conflicts) → the preview flags the risky chords; the operator decides.

---

## Shortcut actions

### US-027: Navigate sessions, desktops, and attention by keyboard

**As an** operator, **I want** first-class actions for session cycling (next/previous), jump-to-attention, focus-last-window, desktop switching by number, desktop creation, and sidebar toggle, **so that** every navigation my hands do in a terminal multiplexer works in the shell.

Acceptance criteria:

- AC-1: Session cycle next/previous moves focus through the sidebar's visible session order, wrapping at the ends.
- AC-2: Jump-to-attention focuses the session with the most recent unresolved attention (needs-you first, else newest `done`), switching workspace if that session lives elsewhere.
- AC-3: Focus-last-window returns to the previously focused window (toggle behavior on repeat).
- AC-4: Desktop switch 1..9 activates the numbered desktop; desktop create adds one and focuses it; sidebar toggle shows/hides the sidebar — all rebindable like every other action.

Edge cases:

- EC-1: Jump-to-attention with zero attention → calm no-op feedback (nothing focused, brief hint), never an error dialog.
- EC-2: Session cycle with zero or one session → no-op / stays put.
- EC-3: Desktop switch to a number that does not exist → no-op with hint (parity with existing tab-jump behavior).
- EC-4: Actions fired while a modal/editable field has focus → shell actions respect focus rules; typing in inputs is never hijacked.
- EC-5: Attention resolves between keypress and focus → lands on the next-best attention target or no-ops honestly.

### US-028: Trust the cheatsheet and rebind everything

**As an** operator, **I want** the `?` cheatsheet derived from the live shortcut registry (defaults + my overrides + preset) and every action group — tabs included — rebindable in Settings, **so that** what the cheatsheet shows, the settings can change, and both always match reality.

Acceptance criteria:

- AC-1: Pressing `?` outside editable fields opens the cheatsheet; every listed chord matches the effective binding (custom overrides included) at open time.
- AC-2: The tab action group appears in the Settings rebind table (all its actions rebindable), removing today's gap.
- AC-3: Surface-local bindings (palette, permission dock, steer) appear in the cheatsheet as read-only reference rows, clearly separated from rebindable actions.
- AC-4: Modifier glyph rendering is identical everywhere (one shared source of truth).

Edge cases:

- EC-1: `?` typed inside an editable field → inserts "?", never opens the dialog.
- EC-2: An action bound to multiple chords → cheatsheet shows the primary chord with the alternates visible (one canonical surfacing rule).
- EC-3: Cheatsheet opened right after a rebind → reflects the new binding without app reload.

---

## Palette views

### US-029: Navigate nested palette views

**As an** operator, **I want** the command palette to support entering nested views (stack navigation with breadcrumb, backspace-on-empty-query pops one level, dismiss closes all), **so that** scoped pickers live inside the palette instead of spawning new surfaces.

Acceptance criteria:

- AC-1: Given a root palette result that is a view entry, activating it pushes the view: the input clears for view-scoped search, a breadcrumb shows the path, and results now come from the view.
- AC-2: Backspace with an empty query pops one level; with text present it edits the query. Dismiss closes the whole palette regardless of depth.
- AC-3: The mechanism is generic: any built-in registered view behaves identically (search, keyboard navigation, enter-to-activate).

Edge cases:

- EC-1: Nesting three or more levels → breadcrumb stays readable (truncation rule) and pop order is exact.
- EC-2: View with zero results for the current query/filter → honest empty state within the view, root results never bleed in.
- EC-3: Palette closed and reopened → opens at root (no stale stack), unless the design states otherwise explicitly.
- EC-4: View data refreshes while open (live catalog changes) → results update without stealing keyboard selection.

### US-030: Switch sessions through the palette Sessions view

**As an** operator, **I want** a built-in Sessions view with state-filter chips (needs-you / working / finished / idle) and attention-first ordering, **so that** Herdr's navigator flow — filter, pick, land — is two keystrokes away.

Acceptance criteria:

- AC-1: Given the Sessions view is open, sessions of the active workspace list attention-first by default — the Show all option (US-031) widens the list to every workspace — with their state visible per row; typing filters by name/agent; chips narrow by state class.
- AC-2: Activating an entry focuses that session's window and closes the palette; if the session was `done`, arrival marks it seen.
- AC-3: Chips are keyboard-reachable (documented keys), and the active filter is always visible.

Edge cases:

- EC-1: Filter chip with zero matching sessions → empty state names the filter, one keystroke clears it.
- EC-2: Session list changes while filtering (session stops/appears) → the list updates in place; selection stays on the same session when it still matches, else moves to the nearest neighbor.
- EC-3: Workspace with hundreds of sessions → list virtualizes/scrolls; typing stays responsive.
- EC-4: Activating a session whose window was closed → the window opens/restores rather than failing.
