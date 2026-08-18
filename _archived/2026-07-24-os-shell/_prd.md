# PRD: AGH OS Shell (`os-shell`)

> Authoring note: this PRD was written after its `_techspec.md` (inverted order, owner-directed). It formalizes the WHAT/WHY/WHO layer; implementation decisions live in the TechSpec and ADR-001..005 and are deliberately absent here.

## Overview

AGH's category is "a local-first agent operating system", and its public hook is "an open workplace for AI agents" — but the web experience today is a conventional sidebar dashboard showing one page at a time. The category is a claim the product doesn't yet embody, and the one-page-at-a-time model makes AGH's core value — many agents working concurrently — literally invisible: an operator cannot watch two sessions at once.

The OS Shell makes the category literal. The web experience becomes a desktop operating system: a menubar, a wallpapered desktop with free-floating windows, a dock, a sessions rail, a command palette, and per-workspace spaces. Every existing product surface (dashboard, tasks, agents, marketplace, network, and the rest) becomes an application window. The desktop itself becomes durable state owned by the runtime: it restores exactly, follows the operator in real time across every client connected to their daemon — tabs, browsers, or machines pointing at the same daemon — and, like everything in AGH, is inspectable and manageable by agents through structured surfaces.

It is for AGH's operators first (developers supervising real agent work locally), for agents themselves as first-class users of the desktop state, and for runtime developers who need every capability to exist as a documented surface. Its value is a deliberate product bet on identity: as of the dated market research in [analysis/01_analysis_market-research.md](analysis/01_analysis_market-research.md) (2026-07), every surveyed fleet-supervision competitor picked kanban boards, thread lists, agent grids, or manager panels — none fused windows + dock + palette + spaces + live continuity into one local operator cockpit — and the shell is also the first honest answer to observing N concurrent agents side by side. The research equally records the metaphor's known risks (Figma's floating-panels reversal, skeuomorphic-novelty perception); ADR-006/007 carry them as explicit bets with mitigations.

## Goals

Ordered by product priority (ADR-006: identity first).

1. **The category becomes an experience.** After this ships, opening AGH means arriving at a desktop — windows, dock, menubar, spaces — not a dashboard. The "agent operating system" claim is something an operator sees and feels in the first five seconds.
2. **Concurrent work becomes visible.** Operators can open any number of session windows side by side, each streaming live, each independently scrollable and replyable — impossible in the current one-page shell.
3. **Resumption replaces reconstruction.** The desktop restores exactly — window positions, sizes, minimized state, focus, and each window's inner location — across reloads, daemon restarts, and every client connected to the same daemon, converging live. If persistence is unavailable, the shell still works fully and says so truthfully.
4. **Navigation costs approach zero.** ⌘K opens anything from anywhere; every surface keeps a shareable URL; browser back/forward stay meaningful on a desktop.
5. **The desktop is agent-manageable state.** Agents and operators can list, read, write, and watch the desktop arrangement through the same structured surfaces (CLI with structured output, API) that govern every AGH capability — no UI-only state.
6. **Attention stays truthful and glanceable.** Dock badges, the sessions rail's status vocabulary, and the menubar approvals bell surface exactly what the runtime knows — never invented counts (v1 posture per ADR-007).

## User Stories

Full catalog: [_user_stories.md](_user_stories.md).

- US-001–US-003, US-005 — Desktop & windows: opening from the dock, arranging, lifecycle, in-window navigation.
- US-004 — Multi-session observation: the side-by-side value proof.
- US-006, US-008 — Attention: dock badges and the approvals bell.
- US-007 — Sessions rail: scan, filter, open.
- US-009 — Palette & keyboard: ⌘K universal open; runtime picker cedes to ⌘J.
- US-010 — Spaces: one independent desktop per workspace.
- US-011–US-013 — Continuity: exact restore, cross-client convergence, degraded posture.
- US-014 — Compact mode below the desktop breakpoint.
- US-015 — Appearance: wallpaper, reduced motion.
- US-016 — Deep links & URL meaning.
- US-017 — Agent-managed desktop (agent persona).
- US-018 — Documented structured surface (runtime-developer persona).
- US-019 — Menubar: system actions from one predictable home.
- US-020 — Window-scoped dialogs: a confirm never freezes the desktop.
- US-021 — Window snap: halves/quarters in one gesture (ADR-009 amendment).

## Core Features

- **Desktop with floating windows.** Every top-level product surface is an app that opens as a window with macOS-style controls (close/minimize/zoom), draggable by its head, resizable, focus-stacked. Lists and details render inside their app's window with a deepening breadcrumb — drilling in never disturbs the rest of the desktop.
- **Window snap.** Throwing a window against a desktop edge or corner previews the zone and snaps it to that half or quarter in one gesture; snapped windows stay proportional as the viewport changes; every snap action is reachable from the palette and keyboard; agents snap through the same desktop-state surface (ADR-009 pull-in of the ADR-007 follow-up).
- **Multi-instance session windows.** Sessions are the one app where each entity gets its own window; every other app is single-instance (reopening focuses). This is the direct answer to "watch several agents at once".
- **Dock.** The app launcher and attention strip: open/minimized indicators, runtime-backed badges for sessions and tasks needing attention, magnification as polish, one accent "new session" action.
- **Menubar.** Workspace identity and switching, Session/View/Help menus, approvals bell with inline approve/deny, palette trigger, settings.
- **Sessions rail.** A dedicated left rail (not a window): recent and grouped-by-agent views, live status vocabulary (running / waiting / failed / done / idle), filtering, one-click open.
- **Command palette.** ⌘K from any context: apps, sessions, and shell actions with type-ahead. RuntimeSelector hosts such as session creation, agent configuration, and onboarding use scoped ⌘J; active session composers stay truthful to the runtime and do not offer unsupported per-prompt runtime changes.
- **Spaces.** Each workspace owns an independent desktop arrangement — its *space* (windows + wallpaper). Switching workspace swaps the whole bench; a Spaces overview shows thumbnails of every space.
- **Durable, converging desktop.** The arrangement is runtime-owned state: exact restore on reload and daemon restart, live convergence across every client of the same daemon, graceful degraded mode when sync is unavailable — and edits made while degraded are pushed when sync recovers (what you see is what persists).
- **Compact mode.** Below the desktop breakpoint the same windows present as a full-viewport stack with an adapted dock; crossing back restores floating geometry exactly. (Visual design arrives via a dedicated prototype; behavior is specified now.)
- **Appearance.** Per-space wallpapers (ember/mesh/carbon), dock magnification toggle, reduced-motion honored from both the system preference and an in-product toggle.
- **Agent-manageable desktop state.** The desktop arrangement is readable, writable, and watchable through the CLI (structured output) and the API, with stable identifiers, versions, and documented deterministic errors — an agent can arrange the operator's desktop.

Feature interactions that define the product: dock/palette/rail/deep links all converge on the same open-or-focus behavior; the URL always mirrors the focused window; spaces scope everything (windows, wallpaper, attention context); minimize + spaces + palette + zoom together form the declared anti-clutter posture (ADR-007).

## Business Rules

- **Truthful UI is absolute.** No control, count, metric, or status renders unless the runtime backs it. Zero badges are invisible (never "0"); over-cap badge displays cap visibly (e.g. "9+") without lying about zero vs non-zero.
- **Workspace scoping.** A desktop arrangement belongs to exactly one workspace. Nothing from another workspace's space renders in place; following a cross-workspace link switches to the owning space first. Deleting a workspace deletes its space and desktop state permanently.
- **Instance rules.** One window per app; one window per session (a session may only ever have one window; opening again focuses). Opening is idempotent.
- **Navigation truth.** The browser URL always identifies the focused window's current location. Focus changes and in-window navigation both write history; back/forward traverse it step by step.
- **Persistence guarantees.** The arrangement persists per workspace on the operator's daemon; restore is exact and idempotent; continuity spans exactly the clients connected to that daemon. Concurrent edits resolve last-writer-wins per window, with a visible version ordering; different windows never conflict with each other; actions that span multiple records (focusing, closing) apply atomically — no half-applied desktops.
- **Degraded posture.** Loss of persistence/sync never blocks any shell capability; the shell surfaces the degraded condition truthfully and recovers without a reload. On recovery, the arrangement the operator built while degraded is pushed and wins for the windows they touched ("what I see is what persists"); everything else adopts the daemon's truth.
- **Unknown state is dropped, never fatal.** Saved entries that are unreadable, unknown-versioned, or reference retired surfaces are discarded individually; the desktop always boots with everything salvageable.
- **Attention sources (v1).** Exactly three: dock badges (sessions/tasks), rail status vocabulary, menubar approvals bell. Approving/denying applies the decision to the runtime; acting on an already-resolved item yields a truthful error.
- **Limits and defaults.** Desktop breakpoint: 960px. Soft open-window guidance: 12 (a gentle notice, never a hard block). Session status vocabulary: running / waiting / failed / done / idle. Desktop-state writes have a bounded per-entry size and per-workspace quota with documented, deterministic error identifiers.
- **Vocabulary.** `space` = a workspace's desktop arrangement (windows + wallpaper) — a presentation term. `workspace` remains the runtime object. The overview surface is named "Spaces". Desktop state is presentation state, explicitly distinct from agent memory in all public language.
- **Accessibility.** WCAG 2.2 AA against the dark surface ramp; complete keyboard paths for every window action (with palette fallback where browsers reserve shortcuts); reduced motion honored globally, system preference winning over the in-product toggle.

## User Experience

- **First contact.** A new operator lands on an empty wallpapered desktop with one hint: the palette shortcut and the dock. Opening the first window teaches the whole model — everything else behaves like the OS they already know.
- **Daily supervision.** Arrive → the desktop is exactly as left → glance at dock badges and the bell → open/focus the session that needs attention (rail or ⌘K) → windows side by side while agents work → minimize what's done. Switching projects = switching spaces, whole bench at once.
- **Focused work.** Zoom a window to fill the desktop; ⌘K to jump anywhere without touching layout; minimize distractions to the dock.
- **Recovery.** Reload, daemon restart, or opening any other client of the same daemon all land on the same desktop. When sync is down, a quiet indicator says so; work continues, and the arrangement you built while offline persists once sync returns.
- **Small viewports.** Below the breakpoint the desktop becomes a focused stack — same windows, same state, no capability loss.
- **Discoverability & a11y.** Every pointer affordance has a keyboard path; every window action is palette-discoverable; the Help menu lists shortcuts. Status is never conveyed by color alone (shape + text accompany the status vocabulary).

## High-Level Technical Constraints

- **The daemon is the source of truth** for everything rendered, including the desktop arrangement itself. The shell is one client of that truth; it must remain fully local-first with no cloud dependency.
- **Surface parity is mandatory (SD-011).** Any state the desktop persists must be equally reachable through CLI (structured output) and API, with documented deterministic errors — no UI-only capability.
- **Performance from the operator's seat, with a bounded envelope.** With 12 windows open on a developer-class machine, window interactions hold 60fps-class fluidity; desktop restoration completes well under a second; three concurrent clients converge without visible thrash; live streams in multiple session windows do not degrade each other; minimized windows cost nothing perceivable. One measured browser scenario guards this envelope.
- **Continuity reliability is existential.** Market evidence: continuity that fails occasionally is worse than none — hence the exact-restore guarantee, the convergence rule, and the honest degraded mode.
- **Privacy.** Desktop state lives with the operator's daemon; nothing about the arrangement leaves the operator's machines.
- **Existing product surfaces are preserved, not rewritten.** The shell rehosts today's views as windows; their behavior contracts (lists, details, streams) carry over.

## Non-Goals (Out of Scope)

- **A mobile-first product.** Compact mode is graceful adaptation, not a phone-first redesign.
- **Tiling window mode, snap-layout editor / custom zones / multi-zone span / linked seam resize, and auto-arrange ("tidy").** Edge snapping (halves/quarters with zone preview) was pulled into v1 by ADR-009; these deeper arrangement features remain named follow-ups per ADR-007/ADR-009.
- **A first-class attention queue** aggregating approvals + waiting sessions + reviews. Named follow-up per ADR-007; v1 ships bell + badges + rail.
- **Extension-facing desktop-state surfaces.** V1 exposes only the desktop consumer; any future extension-facing state surface requires its own TechSpec and public contract.
- **Agent-native desktop tools.** Agents use the CLI/API surfaces in v1; dedicated native tools are a later decision once demand is observed.
- **Multi-operator collaboration semantics.** Concurrent clients converge last-writer-wins; there is no presence, locking, or merge model for multiple humans co-editing one desktop.
- **Window popout to separate browser/native windows.** Later exploration.
- **Marketing repositioning.** This PRD changes the product; public positioning language changes ride the existing copy system (see Open Questions).

## Architecture Decision Records

Product decisions (this PRD phase):

- [ADR-006: The desktop metaphor is the product identity of the AGH web experience](adrs/adr-006.md) — identity-first goal hierarchy; `space` vocabulary; market risks carried as explicit bets.
- [ADR-007: v1 operator-load posture](adrs/adr-007.md) — attention via bell + badges + rail; anti-clutter via minimize + spaces + palette + zoom; queue/snapping/tidy as named follow-ups (snapping since pulled into v1 by ADR-009).

Implementation decisions (TechSpec phase + amendments): ADR-001..005, [ADR-008](adrs/adr-008.md), and [ADR-009](adrs/adr-009.md) under [adrs/](adrs/) — shell replacement strategy, window↔route model, window-manager mechanics, desktop-state persistence (+ round-1 hardening amendments), palette shortcut ownership, the desktop-state contract resolution, and the window snap layer (post-approval amendment).

## Open Questions

- ~~Compact-mode visual design~~ — **resolved 2026-07-19**: the prototype's compact block (`os-v2.css` <960px mode) is the visual reference; the compact phase runs Visual Contract Mode against it (peer-review B-009).
- **Public positioning vs term collision** — the market uses "agent operating system" to mean an invisible infra layer; AGH now ships a literal desktop. Whether launch copy leans into "OS", the "open workplace" hook, or a new frame is a `COPY.md`-owned marketing decision, out of this PRD's scope.
- **Attention-queue pull-in trigger** — what observed evidence (e.g. operators reporting attention fragmentation across bell/badges/rail) promotes the ADR-007 follow-up into the next program.
- **Soft window guidance tuning** — 12 is the initial value; real usage may move it.
