# Spec: Herdr Parity (Session Attention · Orchestration DX · Shortcuts v2)

One program, three tightly coupled capability sets that close the daily-driver gap between CompozyOS and Herdr (the terminal multiplexer for coding agents the primary user operates all day): honest session attention with notifications, event-driven session orchestration for agents, and multiplexer-grade keyboard navigation. Analysis package: `analysis/herdr-parity-analysis.md` + `evidence/` (verified against the current branch on 2026-08-15).

---

# Part I — Product

## Overview

**Problem.** An operator running many concurrent agent sessions cannot tell, without opening each one, which session needs them. A session that asked a question looks identical to one doing work; a session that finished while they were elsewhere looks identical to one that has been idle all day. No notification channel exists — no alert, no sound, no title signal — and the attention bell counts only permission waits in the active workspace. Orchestrating agents are worse off: they cannot wait for a sibling session to reach a state (they poll in loops), children cannot wake the parent that spawned them, and several lifecycle actions (spawn, stop, approve, answer) require shell access an agent may not have. Keyboard navigation lacks the actions terminal-multiplexer users live by: cycle sessions, jump to the thing that needs me, switch desktops by number, one cheatsheet that tells the truth.

**Who.** Three personas (detailed in `_user_stories.md`): the **operator** living in the CompozyOS shell with many sessions across workspaces; the **orchestrator agent** managing sibling/child sessions through structured surfaces; the **headless operator** scripting and supervising from a terminal.

**Why it matters.** CompozyOS already has roughly 70% of the plumbing — a canonical per-session state vocabulary, a sidebar dot, a bell, live list refresh, governed spawn, durable events. The missing 30% is exactly what turns "a list of terminals" into an agent operating system you can trust while looking away: attention that is honest and immediate, orchestration that is event-driven, and navigation that keeps hands on the keyboard. Herdr proves daily that this combination is what retains a multi-agent power user.

## Goals

After this ships, all of the following are observable:

1. A session that asked a question or requested permission is distinguishable at a glance — in the sidebar, the bell, the palette, and structured CLI output — from a session doing work, within the live-refresh latency target, and the distinction survives daemon restarts.
2. Work finished while the operator was not looking carries a distinct finished-unseen marker (`done`) until the session is viewed; viewing clears it automatically everywhere; nothing but viewing clears it.
3. The operator is notified on every transition into a needs-you state (question, permission, failure) and — coalesced — on completions, across **all** workspaces, with suppression for the session currently in view; activating a notification lands on that session in one motion, switching workspace when needed.
4. The bell badge and the app/tab title count exactly the sessions that need action; finished-unseen work is visible but never inflates that number.
5. A script or agent can block on "session reached one of these states" with a timeout and deterministic outcomes, instead of polling; a parent agent is woken when a child it spawned settles, fails, or blocks.
6. An agent with no shell can spawn, stop, approve, answer, wait on, and cancel the in-flight prompt of sessions it is allowed to manage, and can send the operator a notification — all with deterministic, structured results.
7. Every shortcut action accepts multiple chords and numbered ranges; the operator's terminal-multiplexer layer applies as a one-click reversible preset; every action group (tabs included) is rebindable; the `?` cheatsheet derives from the live registry.
8. The command palette supports nested views; the built-in Sessions view filters by attention state and lands on a session in two keystrokes.
9. Every session list surface — the sessions sidebar, the palette Sessions view, and the operator CLI listing — offers a "Show all" option revealing sessions from every workspace, grouped and labeled by workspace and reachable in one activation; agent-facing surfaces remain workspace-scoped.

## User Stories

[Full user stories](_user_stories.md) — canonical catalog with acceptance criteria and edge cases:

- **Attention states** — US-001..US-004: distinguishable needs-input/needs-auth, server-derived `done`/seen, failures join attention, CLI visibility and filtering.
- **Attention surfaces** — US-005..US-007: bell sections and jump, title count, attention-first sidebar sort.
- **Notifications** — US-008..US-013: needs-you alerts, coalesced completions, sound, configuration everywhere, system-level escalation, agent-sent notifications.
- **Cross-workspace attention** — US-014..US-015, US-031: aggregation and jump, per-workspace mute, Show all in every session list.
- **Orchestration: wait** — US-016..US-017: CLI wait with state sets/timeouts, native wait tool.
- **Orchestration: tools** — US-018..US-023: native spawn/stop/approve/answer, wake-on-settle, prompt-cancel CLI verb.
- **Shortcuts** — US-024..US-028: chord arrays, ranges, terminal preset, navigation actions, truthful cheatsheet.
- **Palette views** — US-029..US-030: nested view mechanism, Sessions view.

## Core Features

### 1. Session attention states

The per-session state vocabulary gains the missing attention semantics: **needs-input** (session asked a question — new), **needs-auth** (permission wait — existing), and **`done`** (finished while unseen — new, derived only by the system). Failures join needs-input/needs-auth in one **needs-you class**; sessions that report nothing render **unknown** honestly. One vocabulary is shared by every surface — web shell, CLI, and agent tools — with one authority deriving it.

Functional requirements: pending questions and permissions surface as states within the latency target and survive restarts; `done` derivation and clearing follow the seen rules (Business Rules); precedence between simultaneous conditions is total and documented; state is never conveyed by color alone.

### 2. Attention surfaces

The existing sidebar dot, bell, and dock become trustworthy: the bell shows **Needs you** and **Finished** sections with per-row jump (cross-workspace included); the bell badge and the app/tab title carry the needs-you count; the sidebar gains an attention-first sort. Task approval rows keep their place in the bell.

Functional requirements: counts equal the definition in Business Rules at all times; jumps focus the target session's window, switching workspace first when needed; every indicator pairs shape/label with color.

### 3. Notifications

A user-facing notification channel with a layered delivery ladder: in-app toast (default on), single built-in sound (default on, one global toggle), system-level notifications (opt-in, firing only while the app is unfocused/hidden), and the always-on title count. Needs-you transitions notify immediately per session; completions coalesce; the session currently in view never notifies. Notifications fire for every workspace; a per-workspace mute silences delivery without hiding counters. Agents send operator notifications through the same channel, sanitized and rate-limited. This channel is distinct from the existing bridge notification presets (task fan-out to external messengers), which are unchanged.

Functional requirements: triggers, suppression, coalescing, dedup, and mute exactly as specified in Business Rules; activating any notification lands on the source session; every delivery channel renders its true availability (no pretend-armed states).

### 4. Cross-workspace attention

Attention is aggregated across all workspaces the operator owns: bell rows labeled by workspace, cross-workspace counts in badge and title, notifications for background workspaces, and one-motion jumps that switch workspace and focus the session. Every session list surface — sidebar, palette Sessions view, operator CLI listing — carries a **Show all** option widening the list from the active workspace to all of them, grouped and labeled by workspace.

Functional requirements: aggregation respects workspace boundaries as an explicit operator-scope read; Show all is operator-only (agent tools stay workspace-scoped), defaults off, persists per operator, and its rows activate with the standard cross-workspace jump; removed/inaccessible workspaces degrade honestly.

### 5. Session orchestration for agents

The wait verb generalizes from "until stopped" to "until any of these states", with timeouts, deterministic outcomes, and identity pinning — and gains a native tool twin so shell-less agents stop polling. Spawn gains **wake-on-settle** (default on): the parent is prompted when a child stops, fails, or blocks, with observable suppression reasons — mirroring the existing task wake behavior. Native tools close the shell gap for spawn, stop, approve, answer; the orphaned in-flight prompt cancel gets its CLI verb. The attention vocabulary is the wait vocabulary: `done` satisfies idle, and attention never alters orchestration semantics.

Functional requirements: same governance as existing spawn (TTL, caps, permission narrowing); self-action prohibitions (Business Rules); every surface returns structured results with deterministic errors; workspace scoping identical to existing session tools.

### 6. Shortcuts v2 and palette views

The binding grammar gains chord arrays and numbered ranges; new actions cover session cycling, jump-to-attention, focus-last-window, desktop switching/creation, and sidebar toggle; the operator's terminal layer ships as a built-in one-click, reversible **terminal preset**; the `?` cheatsheet derives from the live registry and the Settings table covers every group (tabs included). The command palette gains a generic **nested view** mechanism (built-in registrations only in v1) with the **Sessions view** — state-filter chips, attention-first order, enter-to-land — as its first proof.

Functional requirements: validation diagnoses collisions across arrays and range expansions; preset application previews changes and reverts atomically; jump-to-attention pairs with the notification/bell target selection; palette views are keyboard-first, list-shaped, and honest about empty states.

### Feature interactions

The attention vocabulary is the shared spine: it drives dots, bell, title, notifications, the wait predicate, wake-on-settle reasons, the jump-to-attention action, and the Sessions view filters. Notification click-through, bell row jump, and jump-to-attention converge on one "land on session" behavior. The preset and the new actions depend on the grammar work; the cheatsheet and Settings depend on the registry being the single source of truth.

## Business Rules

**Attention classes and precedence**

1. The needs-you class is exactly: needs-input, needs-auth, failed.
2. The finished-unseen class is exactly: `done`. It never counts toward needs-you.
3. When one session simultaneously has a pending permission and a pending question, needs-auth wins; resolving it reveals needs-input without an intermediate idle flicker.
4. Lifecycle presentation outranks attention presentation: a stopped/archived/failed session shows its lifecycle state; `done` never masks a terminal state.
5. Sessions that report no interpretable activity show unknown — never a guessed state.

**`done` and seen**

6. Only the system derives `done`: a turn completed while that session's window was not focused or the app was not visible. No agent, tool, or client can set or report it.
7. Viewing the session in the shell (window focused, app visible) marks it seen and clears `done` for every client. There is no manual dismiss.
8. Passive reads never mark seen: CLI status/list, agent read tools, notifications, and bell rendering leave `done` intact.
9. Seen state and pending question/permission state survive daemon restarts.

**Waiting**

10. A wait targets an explicit state set; the default settled set is: needs-input, needs-auth, idle, stopped, failed.
11. `done` satisfies idle in every wait predicate; attention state never changes orchestration outcomes.
12. Waits are pinned to the session identity present at wait start; a replacement session never satisfies the original wait.
13. Waits are edge-driven, return immediately when the session is already in a target state, and end in exactly one of: state-reached (naming the state), timeout, session-gone, canceled.
14. Default wait timeout is 300 seconds; unbounded waiting requires an explicit flag; the native tool's timeout is capped (cap value fixed in Part II and discoverable in the tool contract).

**Notifications**

15. Notify on every transition into needs-you, per session, immediately; a 5-second per-session dedup window absorbs flapping.
16. Completions notify coalesced within a 5-second window; a group of one collapses to the single form; a group of zero fires nothing. Needs-you alerts are never coalesced away.
17. Focus suppression: no notification (visual or sound) for the session whose window is focused while the app is visible; counters and surfaces still update.
18. Delivery defaults: toast on; sound on (single built-in sound, one global toggle); system-level off (opt-in) and, when on, firing only while the app is unfocused or hidden; title count always on.
19. Notifications fire for all workspaces; a muted workspace delivers nothing (toast/system/sound) but keeps its bell rows and counts. Unmuting never replays missed notifications.
20. The bell badge and title count equal the cross-workspace needs-you count; the bell lists Finished separately.
21. Activating any notification or bell row focuses the source session's window, switching workspace first when needed; arrival at a `done` session marks it seen.
22. Agent-sent notifications: title required (length-capped at 80), body optional (capped at 240), sanitized, rate-limited to 1 per second per session; the result names a provable outcome truthfully — delivered (reached at least one live operator client), no-client, muted-workspace, or rate-limited. Whether a human saw it is never claimed; rendering decisions (focus suppression) stay client-side per rule 17.

**Orchestration**

23. Wake-on-settle is a spawn-time choice, default on; wakes fire when a child stops, fails, or enters needs-you, and carry child identity and reason.
24. Wake suppression (parent stopped, disabled, self-wake, delivery failure) is recorded with an observable reason — never silent loss.
25. No self-action: a session cannot approve its own permission, answer its own question, stop itself via the tool, or wait on itself.
26. Native lifecycle tools carry the same governance, scoping, and risk gating as their existing operator surfaces; destructive actions remain subject to permission mode and approval policy.
27. Answering/approving an already-resolved request returns a deterministic already-resolved outcome, never an error cascade.

**Shortcuts and palette**

28. A chord binds to at most one action across all bindings, arrays and expanded ranges included; violations are rejected naming both parties.
29. User bindings beat defaults; an empty binding disables the shortcut; the scalar binding form remains valid as a one-element array.
30. Preset application is previewed (old → new, collisions named), atomic, idempotent, and reversible in one step to the exact pre-preset state.
31. The cheatsheet always reflects effective bindings (defaults + overrides + preset) at open time; `?` never fires inside editable fields.
32. Palette views: backspace pops one level only when the query is empty; dismiss closes the whole stack; reopening starts at root; view results never bleed across levels.

**Session list scope**

33. Every session list surface (sessions sidebar, palette Sessions view, operator CLI listing) offers a Show all option revealing all workspaces' sessions, grouped and labeled by workspace; activating a row performs the standard cross-workspace jump. The default is the active workspace; the operator's choice persists per surface.
34. Show all is an operator affordance only: agent-facing session tools and listings remain same-workspace scoped.

## User Experience

**Attention flow (operator).** Working in session A; session B (another workspace) asks a question → toast + sound name B and the question reason; the title shows the count for the backgrounded case → one click: workspace switches, B's window focuses → the operator answers → the state clears everywhere, counters drop.

**Inbox drain (operator).** After a heads-down hour: bell shows "Needs you (2) / Finished (5)" → resolve the two blockers first, then walk the Finished section; each visit auto-clears its `done` marker; the bell empties itself — no dismiss buttons anywhere.

**Fleet orchestration (orchestrator agent).** Parent spawns three workers natively (wake-on-settle on) → continues its own work → wake arrives: worker 2 needs permission → parent approves it natively → parent waits on the settled set for the remaining workers with one call each → collects results; zero polling loops, zero shell.

**Keyboard day (operator).** Settings → apply terminal preset (preview, confirm) → navigate windows/desktops with the muscle-memory layer → jump-to-attention lands on whatever fired last → `?` shows the truth about every binding → palette Sessions view: filter needs-you, enter, land.

**Onboarding and discoverability.** Toast, sound, and title count work out of the box with zero setup; system-level escalation lives in Settings behind an honest permission flow; the preset is one visible control in Settings; `?` is bound by default; the Sessions view is a root palette entry.

**Accessibility.** No state is conveyed by color alone — every indicator pairs a distinct shape/glyph with an accessible label (the signal palette stays informational). Attention jumps manage focus explicitly; all palette/bell/cheatsheet flows are fully keyboard-operable; live announcements for attention changes are polite, not interruptive.

The screen-by-screen change map lives in `_uiux.md`. The visual language for every journey above is locked in `docs/design/opendesign/herdr-parity/` (seven boards + `DESIGN-NOTES.md`); implementation tasks cite those boards as Visual Contract rows.

## High-Level Technical Constraints

- **One vocabulary, one authority.** Every surface (web, CLI, agent tools) renders the same attention state from the same derivation; no surface computes its own variant. Renames/extensions of the vocabulary are hard cuts across all surfaces in one change (greenfield rule).
- **Restart durability.** Pending question/permission states and seen markers survive daemon restarts.
- **Latency target.** Attention transitions are visible on live clients within ~2 seconds of the underlying event.
- **Transition truth is server-side.** Clients cannot reconstruct state edges from refresh signals; anything that fires "on transition" must be decided where the transition happens.
- **Agent/operator manageability outcome.** Everything this program ships is inspectable, configurable, and operable without the web UI: attention states and filters in structured CLI output; wait/spawn/stop/approve/answer/cancel/notify as agent-callable surfaces with deterministic errors; notification settings that round-trip settings UI ↔ configuration file ↔ CLI identically.
- **Extension ecosystem expectation.** Attention transitions become observable to runtime extension points (the existing typed hook fabric), and agent-sent notification ability is part of the agent contract (bundled skill docs update accordingly). Palette views are deliberately **not** third-party extensible in v1; the mechanism must not preclude it later.
- **Workspace isolation.** Cross-workspace aggregation is an explicit operator-scope read surface; agent surfaces stay same-workspace scoped exactly as today.
- **Don't regress existing leads.** Command palette speed, governed spawn, durable event log, input queue semantics, layout undo/redo, and byte-identical CLI/programmatic parity all stay intact.
- **Truthful UI.** No control renders for a channel the platform doesn't support; unavailable channels state why.
- **Language and copy.** All public copy follows the product copy spec; new vocabulary (needs-you class, `done`/seen, palette view, terminal preset) lands in the canonical glossary; "notification" is disambiguated from bridge notification presets.

## Non-Goals (Out of Scope)

- **Extension-contributed palette views** — v1 registers built-in views only; the registry design leaves the door open (ADR-003).
- **The terminal layer as product default** — it ships as an opt-in preset; defaults stay collision-safe (ADR-004).
- **A dedicated session-switcher surface** — the palette view covers it (ADR-003).
- **Custom notification sounds and per-session sound mutes** — v1 ships one built-in sound, one global toggle, per-workspace mute only.
- **Cross-instance / network attention** — Compozy Network semantics are untouched; attention is local to one daemon.
- **Wait-on-output / transcript pattern matching** — a later candidate; state-set waiting ships first.
- **Programmable sidebar projections for agents** (Herdr `agent.view.set` analog) — the human-facing sort ships; agent-driven view control does not.
- **Herdr features tracked but not pursued here**: remote-updatable detection manifests, plugin marketplace parity, SSH remote attach, terminal multiplayer, in-app release notes (analysis §5).
- **Worktrees-as-product** — shipped separately (worktree-support program).
- **Task/loop attention semantics changes** — task approval rows stay as they are; the task wake bridge is a pattern reference, not a modification target.
- **Mobile push / email delivery** — the ladder ends at system-level notifications.

## Open Questions

None remain at the product level. All product decisions were resolved in grill rounds 1–3 (2026-08-15) and recorded: scope and MVP order (round 1), attention vocabulary and seen semantics (ADR-001), notification policy, counters, and cross-workspace behavior (ADR-002), palette views and naming (ADR-003), shortcut defaults and preset (ADR-004). Surface-level choices (exact default chords, notification copy, tool/verb naming, timeout caps) are Stage 2 material and land in `_dx.md` / `_uiux.md`.

---

# Part II — Technical

## Executive Summary

Attention becomes a daemon-derived truth with canonical state in the global catalog store: pending clarify/permission live as restart-recoverable pending-interaction records (with denormalized projection columns committed in the same transaction), `done` derives from a per-session seen marker maintained through an operator presence lease, every transition carries a durable ordering timestamp, and every surface keeps consuming `SessionPayload.badge` — one vocabulary, one authority (ADR-001, ADR-005 as amended by review round 1). Transitions are detected at the existing lifecycle/event choke points and published as a new `session_attention_changed` event on the existing catalog stream; clients use it only for ephemeral delivery (toasts/system/sound — focus is client knowledge) and keep refetching state on wake events. Orchestration generalizes the existing stop-wait into a badge-predicate wait (CLI, route, native tool) with transcript-epoch pinning, adds a session wake bridge mirroring `task_wake_bridge`, and wraps existing governed surfaces (spawn/stop/approve/clarify-answer) as native tools. Shortcuts v2 widens the binding grammar (arrays, ranges), moves chord defaults into the daemon (ADR-006), migrates the hardcoded shell chords into the registry, and the palette gains a generic nested-view stack with the Sessions view as first registration. Primary trade-off: one schema migration on `sessions` and a breaking config value-type change, in exchange for restart-honest attention and complete server-side validation — greenfield hard cuts, no compat shims.

## MVP Boundary

MVP = the session-attention seed: attention truth core (badge extension, durable columns, wake-filter widening, transition events, seen route), attention surfaces (tone-map unification, bell sections, title count, sidebar sort + tri-state Show all), and the notification channel (`[attention]` config, settings section, toasts, sound, system escalation, per-workspace mute, `compozy__notify`). Post-MVP, in order: orchestration DX (wait generalization + native lifecycle tools + `notify_creator` + `prompt-cancel`), then shortcuts v2 (grammar, daemon defaults, new actions, migrations, preset, cheatsheet), then palette nested views + Sessions view. Explicitly out of scope: everything in Part I Non-Goals. `cy-create-tasks` assigns task numbers to these phases; the trailing `qa-report`/`qa-execution` pair closes the program.

## Developer Experience

- [Developer experience contract](_dx.md) — golden path, CLI (`wait`, `list` filters, `prompt-cancel`, `notify`, config round-trip), HTTP/UDS routes (`/wait`, `/agent/notify`, `/presence`, settings), `[attention]` config, keyboard defaults + terminal preset, seven native tools, deterministic errors.
- [UI change map](_uiux.md) — 14 surfaces (S1–S14), component plan, locked signal/state mapping. The feature is UI-bearing.
- Visual contract: `docs/design/opendesign/herdr-parity/` — index, `DESIGN-NOTES.md`, seven boards. Tasks 03/05/06 own the `eng-ui-screenshot` evidence bundles. Artboard CSS is a contract, never a stylesheet to import.

## System Architecture

| Component | Responsibility |
| --- | --- |
| Attention derivation (`internal/session`) | Badge extension (`waiting-for-input`, `done`), attention classes, transition detection at choke points, canonical pending-interaction records + seen/ordering projection, operator presence leases |
| Wait service (`internal/session`) | Edge-driven badge-predicate waits with epoch pinning, timeout, caps; consumed by route, CLI, native tool |
| Session wake bridge (`internal/daemon`) | `notify_creator`: child settle/block → synthetic prompt to creator (mirror of `task_wake_bridge`) |
| Notify service (`internal/session`) | Agent/operator notification publish: sanitize, caps, per-session rate limit, truthful outcomes |
| Attention config (`internal/config` + `internal/settings`) | `[attention]` section, live lifecycle, settings section + PATCH route |
| Shortcuts v2 (`internal/windowmanager`) | Binding grammar (arrays/ranges), daemon-owned defaults, full-map conflict validation, effective-keymap serving |
| Native tools (`internal/tools/builtin` + `internal/daemon`) | 7 new descriptors/handlers in new sibling files; toolset, meta rows, binding groups |
| CLI (`internal/cli`) | `wait` generalization, `prompt-cancel`, `notify`, list filters, config classify for new keys |
| Web shell (`web/src/systems/os`, `session`, `settings`, `workspace`) | Attention notifier (edge → toast/system/sound), title hook, bell sections, tri-state Show all, palette view stack + Sessions view, settings pages |

Data flow: session events/lifecycle → badge recompute at choke point → (durable persist) → catalog wake + `session_attention_changed` → web refetch (state) + ephemeral delivery (edges) / CLI-tool waits fire → operator or orchestrator acts → pending columns resolve transactionally → next transition.

## Architectural Boundaries

- No new internal packages. New capability lands as new files in owning packages: `internal/session` (attention, wait, seen, notify, spawn-wake domain types), `internal/daemon` (session wake bridge, new native tool handlers), `internal/tools/builtin` (descriptor file `sessions_attention.go`), `internal/windowmanager` (defaults, grammar), `internal/config` (attention section), `internal/settings` (attention section), `internal/hooks` (attention event family files).
- Import flow stays downward: `session` defines `SpawnWakeNotifier`; `daemon` implements the bridge and injects it (functional option), exactly like `task.WithWakeNotifier`. No package imports `daemon/`, `api/`, or `cli/`.
- Near-cap files must not grow — new siblings instead: `internal/tools/builtin/sessions.go` (455L) → `sessions_attention.go`; `internal/daemon/native_tool_session_calls.go` (341L) → `native_tool_session_attention_calls.go`; `internal/cli/session.go` (338L) → `session_wait.go` (wait moves out), `session_notify.go`; `internal/cli/client.go` → `client_session_wait.go`, `client_session_attention.go`; `web/src/systems/os/hooks/use-os-command-palette.ts` (343L) → view-stack hook in a new file.
- `internal/api/core` stays the canonical handler home; HTTP/UDS only register (route parity tests updated in the same change). `mage Boundaries` needs no new rules (no new packages).

## Implementation Design

### Core Interfaces

```go
// internal/session/badge.go — extended inputs (hard cut; no aliases)
type BadgeInputs struct {
    State               LifecycleState
    HealthState         heartbeat.SessionState
    Health              heartbeat.HealthState
    Failure             FailureKind
    PendingAuth         bool
    PendingClarify      bool // NEW: pending clarification request
    ActivePrompt        bool
    Stalled             bool
    Unseen              bool // NEW: settled activity newer than the seen marker
    IneligibilityReason string
}

const (
    BadgeWaitingForInput Badge = "waiting-for-input" // NEW
    BadgeDone            Badge = "done"              // NEW
)
```

Precedence (total order, first match): `failed` → `stopped` → `waiting-for-auth` → `waiting-for-input` → `hung` → `unhealthy` → `running` → `done` (idle-eligible ∧ `Unseen`) → `idle` → `unknown`.

```go
// internal/session/session_attention.go
type AttentionClass string

const (
    AttentionNeedsYou AttentionClass = "needs-you"
    AttentionFinished AttentionClass = "finished"
    AttentionNone     AttentionClass = "none"
)

func ClassForBadge(b Badge) AttentionClass

type AttentionEvent struct {
    SessionID   string
    WorkspaceID string
    From, To    Badge
    Class       AttentionClass
    At          time.Time // durable ordering key == sessions.attention_changed_at
}
```

```go
// internal/session/session_pending.go — canonical pending-interaction record
type PendingInteractionKind string // permission | clarify

type PendingInteraction struct {
    InteractionID     string // daemon-owned identity (ULID) — the primary key
    SessionID         string
    Kind              PendingInteractionKind
    ProviderRequestID string // process-local id; unique per (session, kind) among pending only
    TurnID            string // staleness detection: dead turn → orphaned resolution path
    Title             string // sanitized + redacted, ≤200
    PayloadJSON       []byte // allowlisted shape (choices []string ≤120 each / closed decision set), redacted, ≤4KB
    Status            string // pending | orphaned | resolved | timed_out | canceled
    Resolution        string // decision / answer summary — sanitized + redacted, ≤240
    ResolvedBy        string // actor kind:id
}
```

```go
// internal/session/session_presence.go — per-client operator presence lease (in-memory)
// Acquire returns an opaque lease id bound to the calling client; renew/release require it.
// Leases are keyed (session_id, lease_id); a client can only renew or release its own lease;
// settle evaluation takes the lease map lock, so lease liveness and the settle transition
// are decided atomically (linearized) — any live lease at settle marks seen, never done.
func (m *Manager) SessionPresence(ctx context.Context, sessionID, leaseID string, visible bool) (string, error)
```

```go
// internal/session/session_wait.go
type WaitRequest struct {
    SessionID string
    Until     []Badge       // empty → settled set
    Timeout   time.Duration // capped by WaitTimeoutMax; the service is ALWAYS bounded
    Epoch     int64         // 0 → pin to the epoch observed at start
    ResumeID  string        // resume a prior wait's live registration (gapless continuation)
}

type WaitOutcome struct {
    Outcome  WaitResult // state-reached | timeout | session-gone | canceled | overflow
    State    Badge
    Waited   time.Duration
    Revision int64  // sessions.attention_revision at outcome time (the fence)
    ResumeID string // on timeout: registration kept alive for a 10s grace; resume gaplessly
}

func (m *Manager) WaitForBadge(ctx context.Context, req WaitRequest) (WaitOutcome, error)

// internal/session/session_seen.go — the ONLY seen-mutation path (driven by presence)
// Sets last_seen_revision := attention_revision in one canonical transaction.
func (m *Manager) MarkSessionSeen(ctx context.Context, sessionID string) error
```

```go
// internal/session/spawn_wake.go — domain side; daemon bridge implements
type SpawnWakeReason string // stopped | failed | needs_attention

type SpawnWakeEvent struct {
    ChildSessionID string
    ChildAgentName string
    Reason         SpawnWakeReason
    Badge          Badge
    Detail         string
    WakeEventID    string
}

type SpawnWakeNotifier interface {
    WakeSpawnCreator(ctx context.Context, creatorSessionID string, ev SpawnWakeEvent) error
}
```

```go
// internal/session/session_notify.go
type NotifyRequest struct {
    SessionID   string // sender (trusted scope)
    WorkspaceID string
    Title       string // ≤80 after sanitize
    Body        string // ≤240 after sanitize
}

type NotifyOutcome string // delivered | no-client | muted-workspace | rate-limited (daemon-provable only)

type NotifyResult struct {
    Outcome      NotifyOutcome
    RetryAfterMS int64 // set only for rate-limited
}

// Delivery transport: a sanitized `operator_notification` named event on the catalog stream
// {notification_id, session_id, workspace_id, title, body, at}. `delivered` is provable:
// the event was handed to ≥1 live operator-client stream subscription at publish time
// (dropped/slow subscribers are closed by the broadcaster and don't count).
func (m *Manager) PublishOperatorNotification(ctx context.Context, req NotifyRequest) (NotifyResult, error)
```

```go
// internal/windowmanager/defaults.go — daemon-owned keymap (ADR-006)
type ShortcutBinding []string // 1..n canonical chords; range families expand

func DefaultKeymap() map[string]ShortcutBinding
func EffectiveKeymap(overrides map[string]ShortcutBinding) (map[string]ShortcutBinding, error)
func CanonicalShortcutsV2(overrides map[string]ShortcutBinding) (map[string]ShortcutBinding, error)
```

`SpawnOpts` gains `NotifyCreator bool` — but every wire/CLI/tool input carries it as an **optional presence-aware value** (`*bool` on request structs; CLI tri-state via flag-changed detection) so omission (→ default true) is distinguishable from an explicit `false`; normalization happens exactly once at the governed spawn boundary, then a concrete bool flows downstream (round-2 B-111). `acp.PromptSyntheticMeta` gains the spawn-wake reason fields following its `Normalize()/Validate()` pattern.

### Data Models

Canonical attention state lives in the **global catalog store** — the same physical database as `sessions` — so every transactional invariant below holds inside one transaction. Per-session event stores remain projections, never authority (existing doctrine); this is the round-1 B-001/B-002 resolution.

New side-table `session_pending_interactions` — the canonical, restart-recoverable record of every pending question and permission request (side-table-vs-JSON: identity/kind/status/turn are matchable columns; `payload_json` is display-only content — never matched on — with an **allowlisted, bounded, secret-redacted shape**):

| Column | Type | Purpose |
| --- | --- | --- |
| `interaction_id` | TEXT PK | Daemon-owned identity (ULID) — provider request ids are process-local and never globally unique |
| `session_id` | TEXT FK | Owning session |
| `kind` | TEXT | `permission` \| `clarify` |
| `provider_request_id` | TEXT | Provider/broker id; UNIQUE per `(session_id, kind)` among `pending`/`orphaned` rows only |
| `turn_id` | TEXT | Turn binding — a dead turn routes resolution through the orphaned path |
| `title` | TEXT | Sanitized + secret-redacted, ≤200 chars |
| `payload_json` | TEXT | Allowlisted shape only (choices: `[]string` ≤120 chars each; decisions: closed set), ≤4KB, secret-redacted |
| `status` | TEXT | `pending` \| `orphaned` \| `resolved` \| `timed_out` \| `canceled` |
| `created_at` / `resolved_at` | TEXT | Lifecycle timestamps |
| `resolution` | TEXT | Decision / answer summary — sanitized + secret-redacted, ≤240 |
| `resolved_by` | TEXT | Actor `kind:id` |

Cardinality is real: a session may hold multiple pending permissions and clarifies simultaneously; selection is always by explicit id, and the badge derives from counts, not single ids.

`sessions` columns (denormalized projections + seen/ordering/fence state, committed in the **same transaction** as the side-table mutation):

| Column | Type | Purpose |
| --- | --- | --- |
| `pending_permission_count` | INTEGER NOT NULL DEFAULT 0 | Badge input `PendingAuth` = count > 0 (multi-request cardinality) |
| `pending_clarify_count` | INTEGER NOT NULL DEFAULT 0 | Badge input `PendingClarify` = count > 0 |
| `attention_revision` | INTEGER NOT NULL DEFAULT 0 | Monotonic per-session fence, incremented by every canonical attention transaction — the causal authority (transcript sequences are never consulted) |
| `last_settled_revision` | INTEGER NOT NULL DEFAULT 0 | Set to `attention_revision` at each settle commit |
| `last_seen_revision` | INTEGER NOT NULL DEFAULT 0 | Set to `attention_revision` by `MarkSessionSeen`; `Unseen` = `last_settled_revision` > `last_seen_revision` |
| `last_seen_at` | TEXT NULL | Seen timestamp for display/audit |
| `attention_changed_at` | TEXT NULL | Cross-session ordering key for every attention transition — including seen-clears; exposed on `SessionPayload` for bell/sort/palette ordering |

Write ordering: (1) global-catalog transaction (side-table + projection columns + revisions + `attention_changed_at`); (2) transcript event appended to the session store as an observational projection (best-effort, repairable, never authority); (3) **catalog wake + attention publish — seen-clears included**: the event-less `done → idle` transition publishes the normal `session_catalog_changed` wake (so every client's read model refetches) plus the attention event, in that order.

Restart recovery: pending rows survive and stay actionable. Resolving an interaction whose turn is no longer live commits **one** catalog transaction that records the resolution AND inserts the queued prompt into the existing `session_input_queue` (same database — atomic; never a second queue): if the bounded queue would overflow, the transaction aborts, the pending record is untouched, and the caller receives the deterministic, retryable `queue-full` outcome. On success the outcome is `resolved-after-restart`; duplicate resolutions are idempotent on `interaction_id` (already-resolved). A resumed provider that still needs input re-asks under a fresh provider request id.

Migration: next gap-free Goose migration (side-table + columns) + declarative schema updates + `atlas.sum` + sqlc via `make codegen` (`eng-schema-migration`; append-only identity; no boot repair). In-memory only (ephemeral by nature): presence leases, notify rate-limiter buckets, wait registry, coalescing windows.

Config value change (breaking, hard cut): `WindowManagerConfig.Shortcuts` `map[string]string` → `map[string]ShortcutBinding` (string or array in TOML, Herdr-style untagged; range strings expand). Delete the old scalar-only type everywhere in the same change (struct, overlay, contract payload, CLI classifier, web parser) — no dual-shape reading.

New wire payloads (`internal/api/contract`): `SessionAttentionEventPayload{session_id, workspace_id, from, to, class, at}`; `OperatorNotificationEventPayload{notification_id, session_id, workspace_id, title, body, at}` (the notify transport — a named catalog-stream event); `SessionWaitRequest/Response` (with `resume_id` + `revision` fence); `SessionPresenceRequest/Response` (lease id issue/renew/release); `PendingInteractionPayload` (sanitized projection for discovery surfaces); `SessionAttentionSummaryPayload{needs_you, finished, by_workspace[]}` (exact cross-workspace counts); `AgentNotifyRequest/Response` (`outcome`, `retry_after_ms`); `SettingsAttentionPayload{toasts, sound, system, muted_workspaces}`; window-manager settings payload gains `defaults` + `effective` keymap maps. `SessionPayload` changes by badge vocabulary plus one field: `attention_changed_at` (the ordering key — attention class itself still derives from badge); session detail/status payloads embed the sanitized pending interactions.

### API Endpoints

All routes registered on HTTP and UDS with `internal/api/spec` OperationSpecs (route-inventory parity tests updated in the same change); handlers as `BaseHandlers` methods in `internal/api/core`:

- `POST /api/workspaces/:workspace_id/sessions/:session_id/wait` — body-validated state set + required bounded `timeout_ms`; server long-poll on the wait service; timeout is a `200` outcome; `410` when the pinned session disappears. Detached from request lifetime is NOT wanted here — the wait is the request; client disconnect cancels it (explicitly exempt from SD-010, documented).
- `POST /api/workspaces/:workspace_id/sessions/:session_id/presence` — operator-only per-client presence lease: acquire (`{visible:true}` without a lease id) returns `{lease_id}`; renew (`{visible:true, lease_id}`), release (`{visible:false, lease_id}`); TTL 15s, renewed ~5s. Leases are keyed `(session_id, lease_id)` — one client can never release another's; settle evaluation is linearized against the lease map, so a settle under any live lease marks seen atomically and never derives `done` (round-1 B-003, round-2 B-101). `MarkSessionSeen` stays the sole internal seen-mutation path. Agent identity → deterministic `403 agent_scope_denied`.
- `GET /api/workspaces/:workspace_id/sessions/:session_id/interactions[?status=pending]` — sanitized pending-interaction discovery (both kinds), same-workspace; CLI `compozy session interactions <id>`; the same sanitized projection embeds in session detail/status payloads, so `compozy__session_status` inherits agent-side discovery (round-2 B-104).
- `GET /api/sessions/attention-summary` — operator-only exact cross-workspace counts `{needs_you, finished, by_workspace[]}` computed by the daemon (never from row pages); the bell badge and title consume this projection, while rows stay cursor-paginated on `GET /api/sessions` ordered by `attention_changed_at` with per-workspace error isolation (round-2 B-113). CLI: `compozy session list --summary`.
- `POST /api/agent/notify` — agent-identity route; `200` with truthful outcome; `422` on cap violations.
- `GET/PATCH /api/settings/attention` — settings section pattern (`window_manager_section.go` exemplar); PATCH validates and applies live.
- `POST /api/agent/spawn` — request gains `notify_creator` (strict decode; default true).
- Catalog stream emits two additional named events: `session_attention_changed` (transition edges — ephemeral delivery only) and `operator_notification` (sanitized agent-notification transport behind `delivered`); wake filter widens to `{permission, clarify, done, error}`; seen-clears publish the normal catalog wake too (Safety Invariant 16).

Caller classification is an authorization boundary, not a handler convention (round-1 B-007): agent identity resolves only through `internal/agentidentity` at agent routes and native dispatch; operator endpoints never infer agent identity. Operator-scope surfaces — presence, the cross-workspace catalog list serving Show all, and the catalog stream's cross-workspace aggregation — deny agent-identity callers with the same deterministic `agent_scope_denied` shape on HTTP and UDS; native session tools never widen beyond `nativeSessionInWorkspace`.

Native tool handlers mirror the same core paths; self-action denials translate domain `ErrPermissionDenied` to `NewToolError(ErrorCodeDenied, …, ReasonApprovalSelfDenied, ReasonPolicyDenied)` exactly like `nativeLoopApproveError`.

## Integration Points

- **Platform notifications**: browser Notification API (web) and the desktop app's notification plugin + capability entry (the only new desktop dependency). Permission state is read back honestly; no retry logic — a denied permission is a rendered state, not an error.
- No other external systems. Bridges, MCP sidecars, and Compozy Network are untouched.

## Impact Analysis

| Component | Impact Type | Description and Risk | Required Action |
| --- | --- | --- | --- |
| `internal/session` badge/catalog | modified | New tokens + inputs + transition publish; medium risk (hot path) | Table-driven derivation tests; publish-after-durable |
| `sessions` schema | modified | 4 columns; low risk (nullable/defaulted) | Goose migration + suites |
| Catalog SSE contract | modified | New event type + widened wake filter; medium (refetch volume) | Codegen + web/E2E mocks co-ship (`eng-contract-codegen-coship`) |
| `[window_manager].shortcuts` config | **breaking** | Value type widens; old scalar-only map deleted | Hard cut across config/overlay/contract/CLI/web in one change |
| Shell shortcut handlers (web) | **breaking** | ⌘K/⌘N/⇧⌘G/⌘[ move to registry | Delete hardcoded handlers + `SHELL_ROWS` |
| Native tool surface | added | 7 new tools; risk-classed, self-action-denied | Descriptors + meta + bindings + skill docs |
| CLI | added/modified | `wait` flags, `prompt-cancel`, `notify`, list filters, config classify | New sibling files; exit codes 75/66 |
| Web shell | added/modified | Bell sections, notifier, title, tri-state, palette views, settings | Per `_uiux.md` component plan + `docs/design/opendesign/herdr-parity/` visual contract |

**Delete targets (no fallback / no compat shim / no placeholder):** `internal/config/window_manager.go` scalar `Shortcuts map[string]string` + its overlay/contract/CLI-classifier mirrors; hardcoded `⌘K/⌘N/⇧⌘G/⌘[` handlers in `use-os-shortcuts.ts` (`isPlainMod` palette/new-session branches); `SHELL_ROWS` in `os-shortcuts-dialog.tsx`; the local tone map in `session-status-line.tsx:13-22` and the duplicate in `session-list-row.tsx:6-12` (one exported dictionary replaces both); the 3× modifier glyph maps (one helper replaces them); chord literals in the TS registry (daemon defaults replace them, ADR-006); single-criterion `isWaitingSession` body (class-based predicate replaces it); old default chords `control+alt+ArrowUp` (zoom), `meta+shift+KeyS` (overview), `control+alt+Bracket*` (desktop switch) — replaced, not aliased. No dual badge vocabularies, no legacy notification keys, no schema fallbacks.

**Compozy Impact Audit:**

- **Native tools**: adds `compozy__session_wait` (Read), `compozy__session_spawn` (Mutating), `compozy__session_stop` (Destructive), `compozy__session_approve` (Mutating), `compozy__session_clarify_answer` (Mutating), `compozy__session_prompt_cancel` (Mutating), `compozy__notify` (Mutating, low); toolset `compozy__sessions` grows; `native_entries.go` rows, binding groups, availability diagnostics, workspace guard on every handler; schema digests regenerate.
- **Extensibility and hooks**: new hook family event `session.attention.changed` (definitions + payload + typed dispatch + introspection + async clone + matchers); extension discovery via existing introspection; palette views explicitly not extension-facing in v1 (registry design leaves the door open, ADR-003); bridge SDKs/MCP sidecars unaffected (checked: no bridge contract or sidecar surface touched; bridge notification presets untouched).
- **Workspace data isolation**: new rows/columns are workspace-scoped session state; cross-workspace aggregation is operator-scope only (existing global catalog routes/stream; events carry `workspace_id`); operator-scope endpoints (presence, cross-workspace listing/stream) deny agent identity with the deterministic `agent_scope_denied` shape on both transports; agent-facing tools/listings stay same-workspace behind `nativeSessionInWorkspace` (Business Rule 34); notify is sender-session-scoped; denial tests prove no cache/SSE path exposes another workspace to an agent identity.
- **Official Compozy skill**: `skills/compozy/references/runtime-operations.md` (wait/verbs/attention vocabulary/interaction discovery), `native-tools.md` (7 tools + boundaries), `window-management.md` (keymap v2, preset, palette views), `configuration.md` (`[attention]`, shortcuts value shape), `tasks-and-orchestration.md` (`notify_creator` for sessions), `desktop.md` (system notifications) — each updated inside the phase that ships its surface (round-2 B-112).

## Extensibility Integration Plan

Added: hook event family `session.attention.changed` (files per the window-manager exemplar: definitions, payloads, dispatch, introspection descriptors, async clone, matcher normalization) — one public payload schema carrying `at`, dispatched post-commit at the owning call site, asynchronous, payload cloned, observational and fail-open (a hook error logs and never blocks or mutates the transition); `compozy__notify` joins the agent contract; effective-keymap discovery via settings API. Changed: none of the manifest/provide/permission surfaces. Explicitly unaffected (checked): extension manifests and provides, skills/capabilities loading, tools/resources registries beyond the new IDs, bridge SDKs, MCP sidecars, Compozy Network protocol docs. Deferred by decision: extension-contributed palette views (ADR-003 — the view registry must not preclude it).

## Agent Manageability Plan

Everything operates without the web UI, consistent with `_dx.md`: attention inspection (`session list --attention/--badge/--all-workspaces`, `session status` badge, structured outputs); waiting (`session wait --until/--timeout/--unbounded`, `POST …/wait`, `compozy__session_wait`); lifecycle (`compozy spawn` with `--no-notify-creator` + `compozy__session_spawn`, `session stop` + tool, `session approve` + tool, `session clarify answer` + tool, `session prompt-cancel` + `compozy__session_prompt_cancel` + route); notifications (`compozy notify`, `POST /api/agent/notify`, `compozy__notify`; config via `compozy config get/set attention.*` and `GET/PATCH /api/settings/attention`); keymap discovery (`GET /api/settings/window-manager` effective map, `compozy config get window_manager`). Deterministic errors per the `_dx.md` Errors table; exit codes 0/65/66/69/75/78.

## Config Lifecycle

Added `[attention]`: `toasts` (bool, true), `sound` (bool, true), `system` (bool, false), `muted_workspaces` ([]string, []) — all `Live` lifecycle class; struct+defaults+validate in `internal/config/attention.go` (worktrees exemplar), overlay `merge_attention.go`, CLI classifier entries, settings section + PATCH, examples in `config-toml.mdx` table + lifecycle matrix, validation tests. Changed `[window_manager].shortcuts`: value type widens to string-or-array with range strings; validation delegates to `CanonicalShortcutsV2`; docs updated; `compozy config set` accepts scalar and array literals. Removed: none (no keys are orphaned; verified `[session.supervision]` and bridge notification presets untouched). Every key ships with docs, examples, validation, and tests in the same change (SD-011).

## Testing Approach

Strategy only — concrete cases live in [`_tests.md`](_tests.md). Unit: table-driven badge derivation (all precedence pairs), attention-class mapping, wait predicate/pinning/timeout math, grammar v2 parse/expand/conflict, preset diff, sanitizer/rate-limiter — fakes only at store/clock boundaries. Integration: migration suites (fresh/reopen/ahead/integrity/equivalence), transactional pending-column coupling, route parity (HTTP=UDS inventories), settings round-trip, wake bridge suppression audit. E2E runtime (Go + acpmock): clarify/permission/done flows driving badge + SSE + wait + wake-on-settle. E2E web (Playwright + MSW fixtures updated with new badges/events): bell sections, toasts (focus suppression), title count, tri-state Show all, palette Sessions view, settings pages, preset apply/revert, cheatsheet truth. Visual language for S1–S14 is proved by the `eng-ui-screenshot` evidence bundles on tasks 03/05/06 against `docs/design/opendesign/herdr-parity/` — an implementation-only screenshot is not parity evidence. Coverage floor 80% per touched package; `make gate` per phase, `make gate-full` at close.

## Development Sequencing

### Build Order

1. **P1 — Attention truth core (daemon + contracts)**: schema migration (side-table + columns) + badge extension + transactional coupling + wake-filter widening + `session_attention_changed` + presence route + hook event — **including** their contract payloads, OpenAPI specs, HTTP/UDS registration + parity tests, and `make codegen`, so P1 closes its own public loop (round-1 N-001). Gate: migration suites + derivation tests + route parity + `make gate`.
2. **P2 — Surfaces of truth**: CLI list filters/`--all-workspaces` + web tone-map unification (S1/S14) + bell sections (S3) + title (S4) + sort + tri-state (S2). Gate: web tests + `make gate`.
3. **P3 — Notification channel**: `[attention]` config + settings section/UI (S8) + attention notifier (toast/sound/system, S5–S7) + `compozy__notify`/CLI/route. Gate: settings round-trip + notifier tests + `make gate`.
4. **P4 — Orchestration DX**: wait service + route + CLI generalization + `compozy__session_wait` + prompt-cancel (CLI verb + `compozy__session_prompt_cancel`) + `session_spawn`/`stop`/`approve`/`clarify_answer` tools + spawn wake bridge with CLI `--no-notify-creator`. Gate: wait/wake integration + acpmock E2E + `make gate`.
5. **P5 — Shortcuts v2**: grammar + daemon defaults (ADR-006) + allowlist growth + config type cut + shell-chord migration + Settings table/recorder (S9) + preset (S10) + cheatsheet (S11). Gate: conflict-validation tests + web tests + `make gate`.
6. **P6 — Palette views**: view stack (S12) + Sessions view (S13) + ⌘E wiring. Gate: web tests + `make gate`.
7. **P7 — Close**: QA scenarios walked (flagged per phase) + `make gate-full`.

Docs ship with their surface, never behind it (round-2 B-112): every phase P1–P6 includes the `skills/compozy/references/*` updates and `packages/site` pages (plus generated CLI/API references) for the public surfaces that phase introduces or changes — each phase is independently mergeable with its human and agent contracts complete. P7 holds only the QA walk and the full gate.

Safe-cleanup separation: deletions listed in Impact Analysis land inside the phase that replaces them, never ahead of it. P2–P3 depend on P1; P4 depends on P1 (vocabulary); P5–P6 are independent of P3–P4 and may parallelize in worktrees.

### Technical Dependencies

No external blockers. Internal: `make codegen` after contract/schema changes; desktop notification plugin added when P3 lands; `mage Boundaries` unchanged.

## Monitoring and Observability

Canonical events with standard correlation keys: attention transitions (already the hook/SSE event; observe ledger keeps the durable trail), wait outcomes (`session.wait.completed` with outcome/duration/predicate), spawn-wake dispatch/suppression (reuse the wake-audit pattern with suppression reason codes), notify outcomes (delivered/no-client/muted-workspace/rate-limited counters via `slog` fields `session_id`, `workspace_id`, `outcome`). Coverage-matrix test asserts each new lifecycle path emits its event. No new alerting thresholds — these are operator-visible signals, not SLOs.

## Technical Considerations

### Key Decisions

- Wait is edge-driven server-side (subscribes to attention transitions), returns immediately on already-satisfied predicates, and is deliberately request-coupled (client disconnect cancels — the explicit SD-010 exemption, because the wait IS the request). Registration is race-free by construction: **subscribe first, then snapshot** — edges arriving between the two are buffered to the subscriber, so a fast transition can never be lost (round-1 B-008). Unbounded waiting exists only as the CLI's transparent repetition of bounded waits, made **gapless** by `resume_id`: a timeout response keeps the server-side registration (and its buffer) alive for a 10-second grace window, and the next request reattaches to it — no inter-request blind spot; an expired `resume_id` returns the deterministic `wait-expired` error and the caller re-registers fresh (round-2 B-108). Per-waiter buffers are bounded (64 edges); overflow ends the wait with the deterministic `overflow` outcome — never silent loss. Session stop/delete fan-out is serialized: the `stopped`/gone edge is delivered to registered waiters **before** their registrations are cleaned up.
- The daemon's focus knowledge is an expiring operator presence lease (15s TTL, ~5s renewal; any client's live lease counts); `done` derives at settle time by consulting the lease atomically with the transition — no client ever has to suppress a wrong daemon badge (round-1 B-003).
- Notify outcomes report only what the daemon can prove — delivery to a live operator client — never rendering: `delivered | no-client | muted-workspace | rate-limited` (round-1 B-006). The proof is concrete (round-2 B-107): notifications travel as the sanitized `operator_notification` named event on the catalog stream; `delivered` means the event was handed to ≥1 live operator-client subscription at publish time (the broadcaster closes dropped/slow subscribers, so liveness is daemon-observable); the web notifier consumes the named event and applies its usual focus/mute rendering rules; the typed result carries `retry_after_ms` for the rate-limited case.
- Seen-clears propagate like every other state change (round-2 B-103): the `done → idle` transition publishes the normal catalog wake (authoritative read-model refetch on every client) plus the attention event — clients never patch state from the attention event alone.
- Native `session_wait` emits activity heartbeats while blocked so supervision never classifies a waiting agent as inactive (SD-001).
- Delivery decisions (toast/system/sound) run client-side where focus is known; the daemon never guesses which client is looking.
- Session cycling freezes its ordering snapshot for a short burst window so `last_activity` re-sorts don't make ⌃⇧↓ non-deterministic; workspace cycling follows the menubar's visible order (one shared comparator) and rides the existing workspace-switch barrier.
- cmdk vim-bindings stay enabled; precedence is "focused UI wins" (K5) — affects only the terminal preset's ⌘⌃ chords while the palette is open.

### Known Risks

- Wake-filter widening raises refetch volume → web coalesces invalidations (existing TanStack behavior); measured in P2 gate.
- Sub-second `done` visibility for a focused viewer → focused client marks seen at settle delivery; Playwright asserts no rendered flicker.
- Workspace cycle resets per-scope worktree selections (existing `setActiveWorkspaceId` semantics) → cycling uses the same barrier; documented; revisit only if QA flags it.
- Browser platform limits (⌃digits tab-switching on non-Apple, sound autoplay) → documented on the shortcuts docs page; truthful-UI states for unavailable channels.

## Safety Invariants

1. Only the daemon derives `done`; `MarkSessionSeen` is the sole seen-mutation path, driven by the operator presence lease; no agent-facing surface may call presence or seen mutation — agent identity is denied deterministically.
2. The canonical pending-interaction row, its `sessions` projection columns, and `attention_changed_at` commit in **one global-catalog transaction**; per-session transcript events are observational projections appended after that commit — the canonical store never depends on them, and no transaction is ever claimed across physical databases.
3. Attention transitions publish only after the canonical global-catalog commit (live-after-durable).
4. A wait never mutates its target; registration subscribes **before** snapshotting so no edge between the two is lost; stop/delete fan-out delivers the terminal edge to registered waiters **before** their registrations are cleaned up; per-waiter buffers are bounded (64 edges) and overflow ends the wait with the deterministic `overflow` outcome — never silent loss; a timeout keeps the registration alive for the 10-second resume grace (gapless `resume_id` continuation), after which it is reaped; registrations are removed on session delete/stop, caller cancel, and grace expiry — no leaked watchers; concurrent waits per session are capped (default 32) with deterministic rejection.
5. Wait pinning: a transcript-epoch or identity mismatch at fire time yields `session-gone`, never a false `state-reached`.
6. Spawn wakes dedupe by `WakeEventID` per cause-episode; suppressions (`wake_creator_disabled`, `session_not_live`, `self_wake`, `delivery_failed`) are recorded, never silent; the bridge checks parent liveness so reaper and wake cannot deadlock.
7. Wake prompt delivery detaches via `context.WithoutCancel`; drain goroutines are owned and joined on daemon shutdown.
8. Notify rate limiting is per sender session, fail-closed to the `rate-limited` outcome; sanitization runs before any persistence or broadcast.
9. Self-action prohibitions (approve/answer/stop/wait on self) are enforced in the domain layer and translated to denied tool errors — never merely hidden by the UI.
10. Shortcut config writes validate the full effective map (defaults + overrides, arrays and ranges expanded) atomically — a rejected write changes nothing.
11. No new secret surfaces: notifications, wakes, and attention events carry no tokens; pending-interaction content (`title`, `payload_json`, `resolution`) passes the existing secret redactor and its size bounds (200/4096/240) before **every** persistence and publication boundary — store, queued prompts, logs, hooks, HTTP/UDS, SSE, native-tool reads; existing `claim_token` redaction rules are untouched.
12. A turn that settles under a live presence lease is marked seen in the same derivation step — `done` is never persisted or published for it; leases are per-client, keyed `(session_id, lease_id)` (renew/release require the caller's own lease id — one client can never release another's), in-memory, TTL-expired; settle evaluation is linearized against the lease map; any client's live lease suffices.
13. `SpawnWakeEvent.Detail` and every wake prompt body pass the wake sanitizer and a 240-character bound before any prompt, log, hook, or stream exposure — no raw secrets, no unbounded text.
14. Operator-scope surfaces (presence, cross-workspace catalog list/stream) reject agent identity with one deterministic `agent_scope_denied` shape on both transports; native session tools never widen beyond `nativeSessionInWorkspace`.
15. Resolving an orphaned pending interaction (dead turn) records the resolution AND inserts the queued prompt in **one** catalog transaction against the existing `session_input_queue` (never a second queue): a would-overflow enqueue aborts the whole transaction, leaves the pending record untouched, and returns the retryable `queue-full` outcome; duplicate resolutions are idempotent on `interaction_id` (`already-resolved`); it never fabricates a live-turn delivery, and the success outcome names `resolved-after-restart`.
16. `done → idle` seen-clears publish the normal catalog wake before the attention event — every client's authoritative read model refetches; no client patches state from the attention event alone.

## Assumptions and Defaults

- Needs-you dedup window 5s per session; completion coalescing window 5s; both fixed (not configurable) in v1.
- Wait: default timeout 300s (CLI and tool), tool hard cap 1800s, API `timeout_ms` required and capped at 1800000; settled set = `waiting-for-input, waiting-for-auth, idle, stopped, failed`; `done` satisfies `idle`.
- Notify: title ≤80, body ≤240 (post-sanitize), 1/s per session; outcomes exactly as `_dx.md`.
- `notify_creator` default true at every spawn surface; wire/tool inputs are presence-aware (`*bool` / flag-changed detection) so omission ≠ explicit false, normalized once at the governed spawn boundary; CLI opt-out `--no-notify-creator`; no config key in v1 (flag-level default only).
- Concurrent waits per session: 32.
- Sound: one bundled asset, no customization; plays once per delivery batch.
- Title count copy finalized in the COPY pass; format carries the needs-you total.
- Preset name: "Terminal"; applies as plain overrides; preview + one-step revert.
- Palette breadcrumb shows at most 3 levels before truncating from the left.
- Session identity for pinning: session id + transcript epoch.
- `session_attention_changed` carries `at` — the durable `attention_changed_at` timestamp (millisecond precision); ordering ties break by session id. No sequence field is promised.
- Presence lease: TTL 15 seconds, renewed every ~5 seconds while the session window is focused and the app visible; per-client, in-memory; acquire returns an opaque `lease_id`, renew/release require it; any live lease counts (multi-client rule).
- Causal authority: `attention_revision` is the only fence — transcript sequences are never consulted for attention truth; `Unseen` = `last_settled_revision` > `last_seen_revision`.
- Wait resume grace: 10 seconds; per-waiter edge buffer: 64; both fixed in v1.
- Pending-interaction bounds: title ≤200, `payload_json` ≤4KB (allowlisted shape), resolution ≤240 — all secret-redacted before storage and exposure.
- Pending-interaction `payload_json` is display-only (choices, allowed decisions); every matchable field is a column.
- Unbounded waiting is a CLI affordance (transparent repetition of bounded waits); the wait service and route are always bounded.

## Web/Docs Impact

`web/`: every surface in `_uiux.md` S1–S14 (systems `os`, `session`, `settings`, `workspace`; generated types refresh via `make codegen`; MSW fixtures/handlers for sessions + os systems updated with new badges/events). Visual contract: `docs/design/opendesign/herdr-parity/` (iterate, never regenerate). `packages/site`: config reference (`config-toml.mdx` `[attention]` + shortcuts value shape), lifecycle matrix, a new shortcuts docs page (default keymap + preset block + platform notes), session/orchestration reference pages for `wait`/`prompt-cancel`/`notify`/`interactions`/`--summary` and the seven tools (generated CLI/API refs regenerate) — each shipped inside its surface's phase. `docs/qa/scenarios/`: new `untested` content-addressed scenarios per changed behavior (flag-then-verify per the QA tracker rule; the task pipeline's QA tail owns the walk).

## Architecture Decision Records

- [ADR-001: Attention state model](adrs/adr-001.md) — two needs-you states + server-derived done/seen; done satisfies idle in waits.
- [ADR-002: Notification policy](adrs/adr-002.md) — focus-suppressed, coalesced completions, cross-workspace delivery, needs-you-only counters.
- [ADR-003: Palette nested views](adrs/adr-003.md) — generic mechanism, built-in only, Sessions view first, name stays "Command palette".
- [ADR-004: Shortcut strategy](adrs/adr-004.md) — grammar first, complete familiar default keymap, personal layer as preset.
- [ADR-005: Attention truth pipeline](adrs/adr-005.md) — durable projection columns, transition events on the existing stream, client-side delivery.
- [ADR-006: Daemon-owned keymap defaults](adrs/adr-006.md) — defaults move to Go; complete conflict validation; effective-keymap discovery.
