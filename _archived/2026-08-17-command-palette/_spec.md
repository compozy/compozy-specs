# Spec: Command Palette — OS-Grade Overhaul

Slug: `command-palette` · Companions: [`_user_stories.md`](_user_stories.md) · `_dx.md` (Stage 2) · `_uiux.md` (Stage 2) · `_tests.md` (Stage 2) · [`adrs/`](adrs/)

Lineage: extends the shipped herdr-parity palette (commit `e596826b`, archived at `.compozy/tasks/_archived/herdr-parity/`). Herdr ADR-003 constraints hold unless explicitly superseded here: the surface keeps the name **Command palette**, nested levels are **palette views**, the nested-view mechanism stays generic, and no dedicated switcher surface exists. ADR-004 of this spec supersedes exactly one herdr constraint (list-only view shapes).

---

# Part I — Product

## Overview

**Problem.** The ⌘K palette shipped as a thin navigation dialog and never grew into the OS surface CompozyOS promises. Commands are hand-written UI fragments with no identity: they cannot be enumerated, ranked, rebound, contributed, or invoked outside the web UI. The OS keeps two disconnected command catalogs — most registered keyboard actions never appear in the palette, and most palette rows cannot be bound to keys. Search is naive substring matching with zero personalization; only one nested view exists; entity search covers three of the OS's ~14 list-bearing domains; extensions have no palette presence; agents have no way to discover or trigger what the operator can.

**Who.** Three personas: the **Operator** (keyboard-first human living in the shell), the **Extension author** (shipping palette presence declaratively, without renderer code), and the **Agent** (an AI session or script that must be able to do everything the operator can, through structured surfaces).

**Value.** This overhaul makes the palette the OS's primary control surface — Raycast-class in coverage, speed, and personalization — backed by one centralized command registry that also powers the keyboard system, the cheatsheet, the menubar, and the agent-facing control plane. One definition per command, projected everywhere; extensible by manifest; manageable by agents. That is the CompozyOS core premise applied to the palette itself.

## Goals

After this ships:

1. **Nothing is out of reach.** Every OS action, app, live entity (across all daemon domains), and settings destination is findable and executable from ⌘K. No command is keyboard-only; no palette row is key-unbindable.
2. **The palette learns.** Ranking blends text relevance with per-workspace frecency and learned query vocabulary; pins and recents make the empty-query state personal. Identical input + state always yields identical order.
3. **Extensions are palette citizens.** An extension declares commands, views (List/Detail/Form/Grid), and default shortcuts in its manifest; they appear on enable, vanish on disable, hot-reload in dev mode, and can never execute code in the operator's renderer or steal the operator's keys.
4. **Agents have parity.** Anything the operator can enumerate or invoke in the palette, an agent can enumerate or invoke through CLI/HTTP/UDS and native tools, under identical policy gates, with structured output and deterministic errors.
5. **One definition, every surface.** Label, chord, availability, and reason render identically in the palette, action panel, menubar, cheatsheet, and settings — derived from one registry, so drift is structurally impossible.
6. **The palette reaches the OS level.** Running under the desktop shell, a global chord summons Compozy from anywhere, and individual commands can carry system-wide hotkeys with honest failure states.
7. **Dead ends become delegation.** A query with no match offers "Ask agent" — the query becomes the opening prompt of a new session, with configurable fallback targets.

## User Stories

Canonical catalog: [Full user stories](_user_stories.md) — 39 stories, 3 personas.

- **Catalog & Search** (US-001..007): full-coverage catalog, fuzzy ranking, entity search across all domains, settings search, empty-query surface, ghost autocomplete, unified scope globe.
- **Context** (US-008): context-aware availability and boosting from daemon truth.
- **Views** (US-009..013): generalized view stack; a curated view per list-bearing domain; Detail, Form, and Grid kinds.
- **Action Panel** (US-014): ⌘K-inside-⌘K secondary actions with meta-actions on every command.
- **Execution** (US-015..017): inline typed arguments, declared destructive confirmation, honest feedback.
- **Personalization** (US-018..021): frecency, adaptive query learning, pins, recents + reset.
- **Keyboard** (US-022..025): rebind-anything, aliases, OS-global hotkeys via the shell, hints + live cheatsheet.
- **NL Fallback** (US-026): "Ask agent" with configurable targets.
- **Extensions** (US-027..031, US-038..039): manifest-declared commands, views (two tiers), default shortcuts; policy inheritance; dev hot reload; programmable-view authoring (React DX) and the operator's live-view contract.
- **Agent Control Plane** (US-032..034): enumerate, invoke, and configure through structured surfaces.
- **Surface Consistency** (US-035..037): registry-derived menubar, destination mode preserved, availability honesty.

## Core Features

### F1 — Unified command registry (full OS coverage)

One registry is the single source of truth for every invokable command: shell/window/tiling/desktop/workspace actions, app launches, view entries, settings destinations, extension commands. Every command has a stable id, title, section/category, icon token, keywords, availability predicate, optional arguments, optional confirmation, and an action. Requirements: complete absorption of both existing catalogs (US-001); registry drives every rendering surface (US-035); ids are permanent and namespaced for extensions.

### F2 — Search and ranking engine

Typo-tolerant matching (normalization, diacritic folding, word-boundary and subsequence matching) over title/alias/keywords, blended with personalization (F6) into a deterministic total order with fixed group precedence and honest sectioning. Ghost autocomplete surfaces the top high-confidence completion (US-002, US-006).

### F3 — Entity and settings search

Every list-bearing daemon domain contributes typed entity results (sessions, tabs, worktrees, agents, tasks, loops, jobs, triggers, bridges, knowledge, vault names, network channels, marketplace, extensions); settings pages/keys are searchable destinations. One scope model (the globe) widens all domains from current-workspace to all-workspaces, persisted through the existing shared preference (US-003, US-004, US-007). Selecting an entity lands with the single canonical landing path per domain (session landing reuses the shipped attention-jump semantics — one path, not two).

### F4 — View system

The shipped view-stack mechanism (push/pop, breadcrumb ≤ 3, ⌫-pops-on-empty, Esc-closes-all, reopen-at-root, selection survival) generalizes to four view kinds — List, Detail, Form, Grid (ADR-004) — and to a registered view for every list-bearing domain, each with domain-appropriate chips, truthful counts, and the shared status-tone/attention grammar (US-009..013). Views are serializable data rendered by host-owned components; extension views use the same contract as built-ins.

### F5 — Action panel

⌘K on any selected row opens a filterable action panel: primary action marked and bound to ⏎, sections, per-action shortcuts, destructive styling. Every command row carries meta-actions (Pin/Unpin, Set alias…, Set shortcut…); entity rows carry domain actions (US-014).

### F6 — Personalization

Frecency (time-decayed usage) plus adaptive query learning (query→launch associations with shorter half-life), pins, and recents — all daemon-persisted per workspace, tolerantly pruned, resettable, and excluded from ever storing secrets or argument values (US-018..021).

### F7 — Keyboard system

Every command is bindable: the settings shortcut table lists the whole registry with source filters; conflicts block with the culprit named and explicit overwrite; aliases give typed vocabulary; effective bindings render on rows, menus, and the live cheatsheet. Under the desktop shell, a global summon chord and per-command system-wide hotkeys register at the OS level with honest failure states (US-022..025). Binding truth stays daemon-owned (herdr ADR-006 continuity).

### F8 — NL fallback

Weak/zero results append an explicit "Ask agent: '{query}'" row; ⏎ opens a new session with the workspace's default agent and the query as opening prompt. Fallback targets are configurable and ordered; nothing leaves the machine before ⏎ (US-026).

### F9 — Extension contribution

The extension manifest gains a palette surface: commands (with a serializable action union — invoke own tool, open own view, navigate to app/route, open URL), views (four kinds, in **two composable tiers**: declarative payload projections available to TypeScript and Go, and live **view programs** — real code reacting to search/chips/selection/forms/actions — TypeScript-only this cycle), and default shortcuts (bind only when entirely free; conflicts dormant and visible; user overrides always win). Contributions are workspace-scoped by instance, appear/disappear with enable/disable/health, validate at build/validate time, and hot-reload in dev mode (US-027..031).

### F10 — Agent control plane

Agents enumerate the full registry (ids, availability + reasons, bindings, argument schemas, flags) and invoke commands by id through CLI/HTTP/UDS and native tools — same dispatch, same policy, structured results, deterministic errors, explicit "no attached shell" semantics for UI-effecting commands (US-032..034).

### F11 — Surface consistency

Menubar menus become hand-curated projections of the registry; destination mode is preserved on the new foundation; availability is honest everywhere — disabled rows carry a structured reason, fully-irrelevant commands are hidden, and the same reason text appears on every surface including agent errors (US-035..037).

### Feature interactions

- F1 feeds every other feature: F2 ranks registry entries; F4 views and F3 providers register through it; F7 binds its ids; F9 extends it; F10 exposes it; F11 projects it.
- F5's meta-actions write F6 (pin) and F7 (alias, shortcut) state.
- F8 renders through F2's result assembly (fallback is a result row, not a special dialog).
- F9 executions flow through the same policy machinery F10 uses — one gate for humans, extensions, and agents.

## Business Rules

1. **One id, one command.** Every invokable row has a stable command id, unique across the registry. Extension ids are namespaced `ext.<extension>.<command>`; a collision rejects the later registration at load with a diagnostic.
2. **Descriptors are data.** Command and view descriptors are serializable; no handler functions cross any boundary. Actions form a closed discriminated union; unknown action or view kinds degrade honestly, never crash.
3. **Policy parity.** Every execution path — palette, action panel, menubar, chord, global hotkey, agent invocation — dispatches the same command through the same availability, risk, trust, approval, and confirmation gates. No surface may bypass another's guarantees.
4. **Membership by enablement, availability by health.** Enabled extension instances contribute descriptors (commands/views/default bindings), scoped to the workspace instance (dev overlay wins). Disable/remove deletes contributions; health never does: an unhealthy or degraded source keeps its last-known validated descriptors listed with `available:false` and a structured source status + reason (BR-8 honesty), and its default bindings deactivate without deleting user overrides.
5. **Keys are the operator's.** Extension default chords bind only when entirely free (no core default, no user override, no earlier extension claim — enable order breaks ties deterministically). Conflicted defaults stay dormant and visible in settings. User overrides always beat shipped defaults, including across extension updates.
6. **Daemon owns binding truth.** Defaults, overrides, aliases, and the effective keymap live in the daemon; UI surfaces render the effective keymap and never hardcode chords (zero chord literals in TS — inherited invariant).
7. **View-stack semantics are fixed.** Push/pop, per-level scoped search, breadcrumb ≤ 3 left-truncating, ⌫ pops only on empty query, Esc closes the whole stack, reopen starts at root, no cross-level result bleed, live refresh never steals selection (nearest-neighbor fallback). All four view kinds obey them.
8. **Availability honesty.** A command is hidden only when fully irrelevant to the context; partially relevant commands render disabled with a structured reason; the identical reason string appears on every surface, including structured agent errors. Labels never embed availability prose.
9. **Personalization is workspace-scoped daemon state.** Frecency, query learning, pins, recents persist per workspace in the daemon; browser storage holds none of it. Stale ids prune tolerantly on read. Secrets, password-typed values, and argument values are never recorded.
10. **Ranking is a total order.** Fixed group precedence first, no cross-group exceptions; deterministic tie-breaking with a score deadband. Identical input and personalization state produce identical order.
11. **Vault values never render** in any palette surface — results, previews, detail panes, or match highlights show names/metadata only.
12. **Nothing pre-sends.** The NL fallback transmits the query only on explicit ⏎. No background classification calls, no speculative sends.
13. **Destructive is declared.** Destructiveness and its confirmation copy are descriptor fields. Operator surfaces render the confirmation (Cancel focused); agent invocations map it to the approval flow. An undeclared-destructive command performing destruction is a spec violation.
14. **One scope model.** Entity results and views default to the current workspace; the globe widens every domain together; the preference persists through the existing shared daemon preference and applies across tabs/devices.
15. **Naming is frozen.** The surface is "Command palette"; nested levels are "palette views" (herdr ADR-003). No "Command Center", no parallel switcher surface.
16. **Non-idempotent commands are single-flight.** A second invocation while one is in flight rejects with "already running"; idempotent toggles execute normally.
17. **Menus curate, registry defines.** Menubar layout (grouping, order) is hand-curated; every item's label, chord, availability, and dispatch come from the registry.
18. **Caps are honest.** Any bounded list (root groups, view rows, grid items) either scrolls through everything or reports the exact overflow ("showing N of M"). Silent truncation is forbidden.
19. **The palette never blocks on a cold daemon.** Opening renders the last-known catalog immediately (stale-while-revalidate); daemon-dependent commands disable with the runtime-unavailable reason until refresh completes.
20. **Session landing is single-path.** Every surface that lands on a session uses the shared attention-jump semantics (restore, workspace switch first, mark done-seen). No second landing implementation may exist.
21. **One frame vocabulary, two composable tiers.** Declarative views and view programs emit the same vocabulary and render through the same host components; a declarative view can appear inside a programmable flow and vice versa. No parallel rendering systems.
22. **View programs run in the extension subprocess with per-client sessions.** Extension view code never executes in the renderer; each attached client opening a programmable view gets its own session (independent search, selection, navigation); extension restart destroys sessions and the host reopens fresh.
23. **Typing never waits on a program.** Host-local events (keystroke echo, selection, scroll, chip highlight, ESC) never round-trip; round-trip events are declared per frame; past the soft budget the previous rows stay visible with a busy indicator; a program that misses the hard ack degrades, and repeated misses circuit-break the view — the palette itself never blocks or blanks.
24. **View programs are TypeScript-only this release.** A Go extension declaring `program: true` fails validate with an actionable message; Go extensions contribute commands, default shortcuts, and declarative views.

## User Experience

Personas and goals live in [`_user_stories.md` § Personas](_user_stories.md). Screen-level states will live in `_uiux.md` (Stage 2); journeys here.

**Journey 1 — resident muscle memory (Operator, daily).** ⌘K (or the global summon from another app, under the shell) → palette opens instantly with Pins, Recents, and context group → type 2-4 chars, ghost completes, ⏎ → effect lands, palette closes. Variants: ⌘K on a row for secondary actions; typed alias for the personal vocabulary path; inline arguments when the command declares them; destructive commands interpose their confirmation.

**Journey 2 — find anything (Operator).** Type an entity's name → typed sections render as domains resolve (commands first, entities as they arrive) → widen with the globe if needed → ⏎ lands with the domain's canonical landing (session focus/restore/workspace-switch; app route for others). No knowledge of which app owns what is required.

**Journey 3 — browse a domain (Operator).** ⌘K → "Views" (or a bound chord like ⌘E for Sessions) → domain view with chips and truthful counts → filter, preview in the detail pane, act via ⏎ or the action panel.

**Journey 4 — make it mine (Operator).** Any row → action panel → Pin / Set alias / Set shortcut → settings shortcut table for the full keymap (filter by source, record chords, resolve conflicts with named culprits, reset one/all). The cheatsheet always reflects reality.

**Journey 5 — ship palette presence (Extension author).** Declare commands/views/default shortcuts in the manifest → `build`/`validate` catch every declaration error with actionable messages → enable in a workspace: contributions appear, policy-gated → `dev` hot-reloads palette contributions live → disable removes them cleanly.

**Journey 6 — script the OS (Agent).** Enumerate registry with structured output → inspect availability reasons and argument schemas → invoke by id under policy → adjust bindings/aliases through the settings surface → reset personalization when hygiene demands.

**Accessibility.** The palette implements the ARIA combobox pattern (input keeps DOM focus; active row via active-descendant; arrows move the pointer, not focus), dialog focus trap with focus restore to the trigger, and full keyboard reachability of every action including the action panel and form fields. Single-character and user-defined shortcuts satisfy WCAG 2.1.4: remappable, removable, and guarded against firing in editable targets. State is never conveyed by color alone (glyph + label, per the locked design grammar). A visible trigger (menubar palette affordance) always exists — keyboard is never the only path in.

**Onboarding and discoverability.** First open shows curated defaults (never empty). Rows teach chords via badges; the `?` cheatsheet derives from the live keymap; ghost autocomplete teaches command names; "Ask agent" turns failed searches into productive delegation. No helper copy beneath headings — the surface stays self-explanatory per the UI-description rule.

## High-Level Technical Constraints

- **Required integrations (existing systems, by behavior):** the daemon-owned keymap/settings system (binding truth, effective-keymap projection); the extension lifecycle (manifest validation, enable/disable/health, dev overlay, resource-family machinery); tool execution policy (risk, trust, approvals, availability diagnostics); the window-manager command surface; the sessions/attention landing semantics and shared status-tone dictionary (anti-duplication rules from herdr are binding); the settings shortcut table and cheatsheet; SSE/event streams for live catalog invalidation; the Electron desktop shell for OS-level hotkeys.
- **Performance (user-perspective):** palette opens and renders its catalog instantly (no daemon round-trip on the open path); results update as-you-type without visible jank at full catalog scale (hundreds of commands + large entity sets); async entity waves never reorder or steal selection; a cold or reconnecting daemon degrades to stale catalog + disabled-with-reason, never a blank or blocked palette.
- **Privacy and security:** extension palette contributions are data, never renderer code; all extension-supplied text/content is sanitized and length-capped; vault values are structurally unreachable from palette surfaces; personalization never stores secrets or argument values; its persisted state lives only in the daemon, and the palette client receives exactly one authorized projection of it — the authenticated, workspace-scoped rank-signal snapshot the single scorer needs — held in session memory only and never persisted browser-side; NL fallback pre-sends nothing.
- **Agent/operator manageability outcome:** the registry, invocation, keymap, aliases, and personalization state are inspectable and operable via CLI, HTTP, and UDS with structured output and deterministic errors; native tools expose enumeration and invocation to sessions. UI-only control of any part of this feature is incomplete by definition.
- **Extension ecosystem expectation:** palette contribution rides the existing manifest/resource-family machinery (closed families, operator allowlist ceilings, source-tier trust), with build/validate-time schema enforcement and health-gated live projection.
- **Design constraints:** locked palette visual grammar (shell-glass chrome, neutral-plate selection, signal-token glyph roundels, no color-only state); truthful UI over plausible UI — no control renders for a capability the runtime lacks.
- **QA debt:** `docs/qa/scenarios/ET-palette-sessions-view-switch.md` (`blocked-verify`) must be resolved/re-walked within this cycle rather than superseded by a parallel scenario.

## Non-Goals (Out of Scope)

1. **Composer `/` menu unification.** The in-session slash-command menu stays an independent, session-owned system. Decided in grill: distinct concerns — session-conversation commands vs OS commands. The registry's design leaves a future provider bridge possible but ships none.
2. **URL deeplinks for commands** (web URLs / `compozy://` scheme, "Copy deeplink" actions). Deselected in grill round 3. Agent addressability is covered by the control plane instead.
3. **Automatic natural-language classification** (Warp-style local classifier with inline auto-detection). Rejected in favor of the explicit fallback row; revisitable in a future spec.
4. **Background/scheduled palette commands** (Raycast `interval`-style). Scheduling stays owned by the automation domain (jobs/triggers); the palette registry does not become a second scheduler. Confirmed at Stage-1 checkpoint.
5. **Multi-chord key sequences** (`g` then `i`). The daemon chord grammar stays single-chord (with modifier + ranges); sequences would be a new grammar with its own conflict semantics. Confirmed at Stage-1 checkpoint.
6. **A dedicated switcher surface** and **renaming the palette** — inherited herdr ADR-003 decisions, reaffirmed.
7. **Extension code execution in the renderer** — excluded by construction (ADR-002/ADR-004); not a temporary limitation.
8. **Go view programs (MVU library).** A Go retained-tree/MVU DSL for programmable views is deliberately deferred — net-new framework work not worth this cycle (user decision; ADR-009). The frame vocabulary is language-neutral, so Go joins later without redesign; this cycle Go extensions get commands, shortcuts, and declarative views.
9. **Embedded JS engine in the daemon** (pure-Go QuickJS et al.) — researched and rejected for this cycle: breaks npm built-ins for TS extensions and shares the daemon's process fate (analysis/09; recorded fallback, not a path).

## Open Questions

1. **Cap vs virtualization policy per surface** — resolved in Key Decisions (root capped with honest overflow; domain views/grids virtualize beyond the mount cap via `@tanstack/react-virtual`). The weak-match fallback threshold is likewise resolved: named constant `fallback_weak_match_threshold` in `WeightsV1` (served in the rank-signals projection; boundary cases pinned in the scorer suite).

Resolved during Stage 2 surface grill (recorded in `_dx.md`/ADRs): view-contract technology (Go-owned typed vocabulary — research verdict, ADR in Part II), domain identifier `cmd_palette` (ADR-005), global summon default `meta+shift+Space`, fallback v1 = `agent` only, `personalization` master switch kept.

---

# Part II — Technical

## Executive Summary

A new Go domain package, `internal/cmdpalette`, becomes the daemon-canonical command registry (ADR-006): serializable command descriptors (legacy-commander keystone) assembled from core declarations plus a storage-free extension projection (the `internal/extension.Manager.Commands` pattern), content-hash revisioned (the `internal/command.Catalog.Revision` sha256 pattern), served over `/api/cmd-palette/*` on HTTP+UDS with SSE invalidation, and consumed by one web-side projection that renders every surface — palette, action panel, menubar, cheatsheet, settings — from the same client registry. Ranking ports the supercmd model (frecency + adaptive query learning, ADR-003) into pure Go with personalization persisted in globaldb; text matching and volatile-context evaluation stay client-side for keystroke latency. Views are a fixed, versioned Go-owned descriptor vocabulary rendered exclusively by host components, with RFC 6902 JSON Patch live updates (ADR-007). OS-global hotkeys land through a new product-window preload bridge in the Electron shell (`desktop/`), which today has no renderer IPC at all.

Primary trade-offs: a large one-time absorption migration (every hardcoded palette group, the TS action mirror, and the closed keymap enums are delete targets — greenfield hard cut), and ownership of a small rendering-engine contract (view vocabulary + patch semantics) in exchange for extension views that are wire-data-only in the renderer, keyboard-perfect, and design-token-faithful by construction.

Ranking has exactly one executable implementation: a pure TypeScript scorer on the keystroke path (headless-tested, golden-fixture-pinned), fed by daemon-served personalization signals and versioned weights (ADR-003 as amended) — no Go scorer, no dual maintenance. Extension views ship in two composable tiers (ADR-008/009): a declarative tier (tool payload → frame projection, both languages) and a programmable tier — live TypeScript view programs executing in the extension's own subprocess over a new `view.provider` capability on the existing provides-RPC channel, using the Raycast/Vicinae event model (stable handler ids, one host→program event kind, event-countered controlled values, frame-clock-batched patches, per-client sessions, three-band latency budget with circuit-breaking). The `@compozy/extension-react` package carries the React layer (react-reconciler in persistent mode); the Go MVU equivalent is deferred (ADR-009).

## MVP Boundary

The MVP is slices P1–P4 of the Build Order: the registry vertical slice (full core absorption with its complete agent surface — API, CLI, native tools, docs — co-shipped), personalization, execution UX, and the keyboard slice (rebind-anything + aliases + mutation verbs). Post-MVP inside this spec: P5 view system, P6 extension contribution (+ view programs), P7 desktop global hotkeys, P8 NL fallback + QA close; P0 (design artboards) precedes any UI-bearing slice. Out of scope: the Part I Non-Goals.

## Developer Experience

- [Developer experience contract](_dx.md) — manifest (`resources.cmd_palette`), CLI (`compozy cmd-palette list|inspect|invoke|personalization`), HTTP/UDS (`/api/cmd-palette/*`, settings PATCH extensions), config (`[cmd_palette]`, summon chord), native tools (`compozy__cmd_palette_*`), deterministic errors.
- [UI change map](_uiux.md) — 16 surfaces (S1–S16), states per story, component plan, signal mapping.

## System Architecture

| Component | Responsibility | Boundary |
| --- | --- | --- |
| `internal/cmdpalette` (new) | Descriptor/Action/View types, catalog assembly + sha256 revision, personalization signals + versioned-weights authority (scoring runs client-side — Key Decisions), context-key contract, invoke orchestration | Pure domain; no api/http imports |
| Core command declarations | Every core command as a `Descriptor` (window/tiling/desktop/workspace/session/app/settings-destination), replacing the closed action enums as the id source of truth | Declared in `internal/cmdpalette/corecmds` (split per section, 500-line cap) |
| `internal/extension` additions | `resources.cmd_palette` manifest block + validation; membership projection `Manager.CmdPalette(workspaceID)` (enable-scoped; health feeds availability/source status, never membership — last-known validated descriptors retained) | Extension package owns manifest types; cmdpalette consumes via a `Provider` interface |
| Keymap evolution (`internal/windowmanager`) | Bindable-id space opens to the registry (core + `ext.*`); extension-default tier (bind-only-if-free); `palette.summon.global` default | Validation delegates known-id checks to the registry |
| Personalization store | globaldb tables (usage, query hits, pins) via sqlc; snapshot API for client ranking | `internal/store/globaldb` fragment `44_cmd_palette.sql` |
| API layer | `/api/cmd-palette/{commands,commands/{id}/invoke,personalization,usage,stream,views/*}` + settings sections; OpenAPI + generated TS; SSE `cmd_palette.catalog.changed` | `internal/api/core/cmd_palette*.go`, routes on both transports |
| Web projection (`web/src/systems/os/`) | Registry hydration (SWR, revision-keyed), client context evaluator, one dispatch seam (client-op vs daemon-invoke), search/rank blending, all render surfaces from `_uiux.md` | Replaces the four hardcoded group components + god hook + dispatch if-chain |
| Desktop shell (`desktop/`) | Product-window preload bridge (`window.compozyShell`), `globalShortcut` module with typed failure statuses, summon event, accelerator conversion | New `desktop/src/product/` + `desktop/src/shortcuts/`; contract module modeled on `boot-contract.ts` |
| View-program runtime | `view.provider` capability + `view/open|event|close` service calls, daemon view-session registry (clarify-runtime generalization, per-client), `view/patch` host-API push → SSE fan-out, protocol additions (cancel frames, negotiated view timeout, view rate class, caps) | Daemon: `internal/extension/view_source.go` + `internal/cmdpalette` session types; transports per analysis/08 seams |
| React kit package | `@compozy/extension-react` (new workspace package, `sdk/react/`): palette component kit (List/Detail/Form/Grid/ActionPanel/Action + hooks), react-reconciler persistent-mode → frames/patches, starvation guard; depends on `@compozy/extension-sdk` + generated vocabulary typings | `sdk/react/src/*`; `registerViewProvider` + view-session registry stay in `sdk/typescript/src/view-provider.ts` (seams in analysis/08 appendix) |
| CLI + native tools | `compozy cmd-palette *` verbs; `compozy__cmd_palette_list/invoke` descriptors + catalog entry | `internal/cli/cmd_palette*.go`, `internal/tools/builtin` |

**Data flow.** Core declarations + extension projections → catalog(revision) → GET commands / SSE changed → web hydration → surfaces. Invoke: surface → dispatch seam → client op (window/nav/overlay via existing coordinators) or daemon invoke (tool policy/approvals) → result contract → feedback + usage recording → personalization → next snapshot. Views: built-ins construct the normalized view model in TS from their TanStack queries; extension views deserialize daemon-validated payloads into the same model; live updates arrive as revision-fenced JSON Patch.

**Story → component map.** US-001..008 → registry core + web projection + context evaluator; US-009..013 → view vocabulary + generalized shell + domain views; US-014..017 → action panel, args bar, confirmation, dispatch/feedback; US-018..021 → ranking engine + personalization store + snapshot API; US-022..025 → keymap evolution + settings table + cheatsheet derivation; US-026 → fallback assembly + session-spawn integration; US-027..031 → extension manifest/projection/dev-reload; US-032..034 → API layer + CLI + native tools; US-035..037 → menubar projection + destination re-source + availability contract; US-038..039 → view-program runtime + `@compozy/extension-react` + the web degradation/band UX. Every `_dx.md` surface maps to the API layer/CLI/tools rows; every `_uiux.md` surface maps to the web projection row.

## Architectural Boundaries

- New package `internal/cmdpalette` (+ `internal/cmdpalette/corecmds`). Import direction: `api/core` → `cmdpalette` → (`store/globaldb` sqlcgen, `windowmanager` id/chord helpers). `cmdpalette` never imports `api`, `httpapi`, `udsapi`, `cli`, or `extension`.
- `internal/extension` defines manifest types (`CmdPaletteConfig`) and the projection; `cmdpalette` defines `Provider` and receives the extension projection via the `daemon/` composition root (`boot_resource_graph.go` peer wiring) — no cycle.
- `internal/windowmanager` keeps chord grammar/canonicalization; known-id validation moves behind a `BindableIDs` source supplied by `cmdpalette` at composition time.
- Web: `web/src/systems/os/` owns projection/dispatch/surfaces; `packages/ui` gains nothing unless the design pass proves a generic primitive (reuse-before-create rule); no palette state outside the OS system.
- Desktop: `desktop/src/product/product-contract.ts` is the single IPC contract; renderer access only via the preload bridge; `sandbox:true`/`contextIsolation:true` unchanged.

## Implementation Design

### Core Interfaces

```go
// internal/cmdpalette/descriptor.go
type Descriptor struct {
    ID           string        // stable, unique; ext.<name>.<cmd> for extensions
    Title        string
    Section      string
    Icon         string        // lucide token or single emoji grapheme
    Keywords     []string
    Source       Source        // {Kind: core|extension, Extension string}
    Action       Action
    Arguments    []Argument    // inline-arg schema (text|password|dropdown|checkbox)
    Destructive  bool
    Confirmation *Confirmation // required when Destructive
    When         []Predicate   // closed context keys (shared evaluator; see Key Decisions)
    AvailabilityExempt bool    // survives daemon reconnect (e.g. cheatsheet)
    Policy       ExecutionPolicy // {SingleFlight bool; RetrySafe bool} — DECLARED per command
    // (defaults: tool/destructive → SingleFlight, RetrySafe false; navigate/url → parallel,
    // retry-safe); Action.Kind cannot derive idempotence — the field is authoritative and
    // projected on every wire/list/inspect surface (round-2 B-007).
    // Execution site is DERIVED from Action.Kind (tool → daemon; client_op|navigate|
    // url|view → client) — no separate field; validation rejects impossible pairs (N-004).
}

type Action struct {
    Kind ActionKind     // client_op | tool | view | navigate | url
    Op   string         // client_op: window.close, overlay.sessions, ...
    Tool string         // tool: fully-qualified tool ID
    View string         // view: registered view id
    App  string         // navigate: OS app id
    URL  string         // url: external
    Args map[string]any `json:"args,omitempty"` // bound arguments (row actions, deep invocations)
}
```

```go
// internal/cmdpalette/registry.go
type Registry interface {
    Catalog(ctx context.Context, ws workspace.ID, client ClientID) (Catalog, error)
    Invoke(ctx context.Context, req InvokeRequest) (InvokeResult, error)
    RecordUsage(ctx context.Context, u Usage) error
    Personalization(ctx context.Context, ws workspace.ID) (Snapshot, error)
    ResetPersonalization(ctx context.Context, ws workspace.ID) error
    Pin(ctx context.Context, ws workspace.ID, commandID string) error   // idempotent
    Unpin(ctx context.Context, ws workspace.ID, commandID string) error // idempotent
}

type Provider interface { // implemented by corecmds and the extension projection
    ProvideCommands(ctx context.Context, ws workspace.ID) ([]Descriptor, error)
    // typed canonical IDs at every daemon boundary; raw ID/name/path strings exist
    // only at the public resolver (L-033 — Impact Audit).
}

type Catalog struct {
    Commands []ResolvedCommand // Descriptor + Available + UnavailableReason + Bindings + Alias + Source
    Sources  []SourceStatus    // core + one per extension instance
    Revision string            // sha256 over STRUCTURAL state only: descriptors, sources,
    // bindings, aliases — never client-resolved availability (round-4 B-006); responses carry a
    // separate context_revision (the targeted client's windowmanager snapshot revision).
}

type SourceStatus struct {
    Source string       // "core" | "ext.<name>"
    Status SourceHealth // healthy | unhealthy | degraded | disabled
    Reason string       // verbatim operator-facing reason when not healthy
}
```

```go
// internal/cmdpalette/personalization.go — the daemon owns SIGNALS + WEIGHTS, not scoring.
// Scoring executes in ONE TypeScript implementation (web/src/systems/os/lib/ranking/,
// pure + headless vitest); golden cross-fixtures pin the algorithm (ADR-003 amendment).
type Weights struct{ /* every constant named + Version; served inside Snapshot */ }

type Snapshot struct {
    Usage     []UsageSignal // per-command decayed weight + last_used_at
    QueryHits []QueryHit    // normalized query → command associations
    Pins      []Pin
    Weights   Weights
    Revision  string
}

func DecayFrecency(w float64, last, now time.Time, halfLife time.Duration) float64 // snapshot maintenance
// TS module contract (single scorer): rank(query, candidates, snapshot) → total order
// (fixed group precedence, deadband, transitivity property) + assembleSections(floors).
```

```go
// internal/tools/approval_async.go — the canonical approval owner's async contract
// (round-3 B-002; durable model: tool_approval_pending, migration 00069)
type ApprovalCoordinator interface {
    Begin(ctx context.Context, req ApprovalRequest) (ApprovalTicket, error) // mints stable public ApprovalID; persists pending
    Resolve(ctx context.Context, approvalID string, outcome ApprovalOutcome) error // fenced terminal transition; resumes exactly once
    Status(ctx context.Context, approvalID string) (ApprovalStatus, error)
    Cancel(ctx context.Context, approvalID string) error // initiator withdrawal → terminal canceled
}
// Recovery: boot re-arms expiry for pending rows (expired → timeout); the approved→resumed
// step claims resume_fence atomically — a restart can never double-execute the provider.
// The palette consumes this; approval surfaces (HTTP/UDS/CLI/native) co-ship in P1.
```

```go
// internal/cmdpalette/view.go — JSON tags ARE the frozen _dx.md wire (ADR-007).
// Generated TS types derive from these structs; round-trip fixtures cover every
// _dx.md example (UT-040 family). No internal normalized second schema.
type ViewPayload struct {
    View     string      `json:"view"`               // contract version: "v1"
    Chrome   *ViewChrome `json:"chrome,omitempty"`   // loading/search/pagination/handler slots
    Sections []Section   `json:"sections,omitempty"` // kind list
    Chips    []Chip      `json:"chips,omitempty"`
    Empty    *EmptyState `json:"empty,omitempty"`
    Detail   *DetailBody `json:"detail,omitempty"`   // kind detail
    Form     *FormBody   `json:"form,omitempty"`     // kind form
    Grid     *GridBody   `json:"grid,omitempty"`     // kind grid
}
type Section struct {
    Title string `json:"title,omitempty"`
    Rows  []Row  `json:"rows"`
}
```

```go
type Row struct {
    ID          string            `json:"id"`
    Title       string            `json:"title"`
    Subtitle    string            `json:"subtitle,omitempty"`
    Icon        string            `json:"icon,omitempty"` // token | emoji
    Badge       *Badge            `json:"badge,omitempty"`
    Keywords    []string          `json:"keywords,omitempty"`
    Accessories []string          `json:"accessories,omitempty"`
    Detail      *DetailBody       `json:"detail,omitempty"` // List.Item.Detail accessory
    Actions     []RowAction       `json:"actions,omitempty"`
    Requires    map[string]string `json:"requires,omitempty"` // adaptive-cards-style
    Fallback    string            `json:"fallback,omitempty"` // element | "drop"
}
type RowAction struct {
    Title        string        `json:"title"`
    Icon         string        `json:"icon,omitempty"`    // token | emoji
    Section      string        `json:"section,omitempty"` // ActionPanel.Section grouping
    Primary      bool          `json:"primary,omitempty"`
    Destructive  bool          `json:"destructive,omitempty"`
    Confirmation *Confirmation `json:"confirmation,omitempty"` // required when Destructive
    Shortcut     string        `json:"shortcut,omitempty"`
    Action       *Action       `json:"action,omitempty"`  // policy-dispatched union (host-executed)
    Handler      string        `json:"handler,omitempty"` // view-local handler id (program-executed)
    SubmitForm   bool          `json:"submit_form,omitempty"` // <Action.SubmitForm>: submits the enclosing form
    Requires     map[string]string `json:"requires,omitempty"`
    Fallback     string        `json:"fallback,omitempty"`
    // Validation: exactly one of Action|Handler|SubmitForm (SI-18); Destructive requires Confirmation.
}
type Chip struct {
    ID    string `json:"id"`
    Label string `json:"label"`
    Count *int   `json:"count,omitempty"`
    Requires map[string]string `json:"requires,omitempty"`
    Fallback string            `json:"fallback,omitempty"`
}
type EmptyState struct{ Title string `json:"title"`; Hint string `json:"hint,omitempty"`; Icon string `json:"icon,omitempty"` }
type Pagination struct{ HasMore bool `json:"has_more"`; PageSize int `json:"page_size,omitempty"` }
```

```go
type DetailBody struct {
    IsLoading bool        `json:"is_loading,omitempty"` // Detail/List.Item.Detail loading state
    Markdown  string      `json:"markdown,omitempty"`   // sanitized host-side
    Metadata  []MetaField `json:"metadata,omitempty"`   // {Label, Value}
    Actions   []RowAction `json:"actions,omitempty"`
}
type FormBody struct {
    Fields   []FormField `json:"fields"`
    Submit   *RowAction  `json:"submit,omitempty"`    // <Action.SubmitForm> maps to SubmitForm:true
    OnSubmit string      `json:"on_submit,omitempty"` // handler id: <Form onSubmit(values)>
}
type FormField struct {
    ID          string   `json:"id"`
    Type        string   `json:"type"` // text|password|textarea|checkbox|dropdown|file
    Label       string   `json:"label"`
    Placeholder string   `json:"placeholder,omitempty"`
    Required    bool     `json:"required,omitempty"`
    Options     []string `json:"options,omitempty"`     // dropdown
    Directories bool     `json:"directories,omitempty"` // file picker mode
    Default     any      `json:"default,omitempty"`
    Error       string   `json:"error,omitempty"`       // field-scoped validation message
    EmptyHint   string   `json:"empty_hint,omitempty"`  // dropdown-with-zero-options copy (US-012.EC-4)
    OnChange    string   `json:"on_change,omitempty"`   // handler id (opt-in dispatch)
    OnBlur      string   `json:"on_blur,omitempty"`
    EventCount  int64    `json:"event_count,omitempty"` // controlled-value fence (ADR-008)
    Requires    map[string]string `json:"requires,omitempty"`
    Fallback    string   `json:"fallback,omitempty"`
}
type GridBody struct{ Sections []GridSection `json:"sections"` }
type GridSection struct{ Title string `json:"title,omitempty"`; Tiles []GridTile `json:"tiles"` }
type GridTile struct {
    ID string `json:"id"`; Title string `json:"title"`
    Image Image `json:"image"` // {URL|Token|Emoji}, resolved before crossing
    Badge *Badge `json:"badge,omitempty"`; Actions []RowAction `json:"actions,omitempty"`
    Requires map[string]string `json:"requires,omitempty"`; Fallback string `json:"fallback,omitempty"`
}
type ViewPatch struct {
    ViewID string    `json:"view_id"`
    From   string    `json:"from"`
    To     string    `json:"to"`
    Ops    []PatchOp `json:"ops"` // RFC 6902; fenced
}
```

```go
// Chrome, handler slots, and the remaining final wire structs — no prop of the
// frozen React-kit inventory lacks a Go representation (round-2 B-004).
type ViewChrome struct {
    IsLoading   bool        `json:"is_loading,omitempty"`
    SearchText  string      `json:"search_text,omitempty"` // controlled; pairs with EventCount
    EventCount  int64       `json:"event_count,omitempty"`
    Placeholder string      `json:"search_placeholder,omitempty"`
    ThrottleMs  int         `json:"throttle_ms,omitempty"`
    Filtering   *bool       `json:"filtering,omitempty"` // explicit override of the presence rule
    Complete    bool        `json:"complete,omitempty"`  // true → host filters locally (LSP model)
    ActiveChip  string      `json:"active_chip,omitempty"`
    Columns     int         `json:"columns,omitempty"`  // grid
    Pagination  *Pagination `json:"pagination,omitempty"` // {HasMore bool; PageSize int}
    OnSearch    string      `json:"on_search,omitempty"` // handler ids; presence flips ownership
    OnChip      string      `json:"on_chip,omitempty"`
    OnSelection string      `json:"on_selection,omitempty"`
    OnLoadMore  string      `json:"on_load_more,omitempty"`
}
type Effect struct { // TYPED closed union — exactly one member set; at-most-once by ID (SI-21)
    ID        string           `json:"id"`
    Toast     *ToastEffect     `json:"toast,omitempty"`      // {Tone, Message}
    Copy      *CopyEffect      `json:"copy,omitempty"`       // {Content}
    OpenURL   *OpenURLEffect   `json:"open_url,omitempty"`   // {URL} — same rules as the url action kind
    OpenApp   *OpenAppEffect   `json:"open_app,omitempty"`   // {App} — same rules as navigate
    PickFiles *PickFilesEffect `json:"pick_files,omitempty"` // {Directories bool}; result → EffectResult
}
type ToastEffect struct{ Tone string `json:"tone"`; Message string `json:"message"` }
type CopyEffect struct{ Content string `json:"content"` }
type OpenURLEffect struct{ URL string `json:"url"` }
type OpenAppEffect struct{ App string `json:"app"` }
type PickFilesEffect struct{ Directories bool `json:"directories,omitempty"` }
type PatchOp struct{ Op string `json:"op"`; Path string `json:"path"`; Value json.RawMessage `json:"value,omitempty"` }
type Badge struct{ Label string `json:"label"`; Tone string `json:"tone"` }
type Image struct{ URL string `json:"url,omitempty"`; Token string `json:"token,omitempty"`; Emoji string `json:"emoji,omitempty"` }
type MetaField struct {
    Label string `json:"label"`; Value string `json:"value"`
    Requires map[string]string `json:"requires,omitempty"`; Fallback string `json:"fallback,omitempty"`
}
type Confirmation struct{ Title string `json:"title"`; Body string `json:"body,omitempty"`; Confirm string `json:"confirm"` }
```

```go
// internal/cmdpalette/view_service.go — the single view authority (composition-root wired;
// every open/event/stream/close/effect transition routes through it — round-3 B-004)
type ViewService interface {
    ResolveView(ctx context.Context, ws workspace.ID, viewID string) (ViewDescriptor, error)
    OpenSource(ctx context.Context, ws workspace.ID, viewID string) (ViewPayload, error) // Tier-1 (read-only tools)
    OpenSession(ctx context.Context, req ViewOpenRequest) (ViewFrame, SessionToken, error)
    AdmitEvent(ctx context.Context, tok SessionToken, ev ViewEvent) error   // seq admission + caps (SI-19)
    PublishFrame(ctx context.Context, tok SessionToken, f ViewFrame) error  // program push (view/patch)
    AckEffects(ctx context.Context, tok SessionToken, effectIDs []string) error
    CloseSession(ctx context.Context, tok SessionToken, reason string) error // idempotent (SI-13)
    InvalidateInstance(ctx context.Context, ws workspace.ID, ext string, generation uint64) error
}
```

```go
// internal/cmdpalette/view_program.go — the view.provider RPC contract (ADR-008)
type ViewOpenRequest struct {
    ViewSession string         `json:"view_session"` // daemon-minted, per attached client
    View        string         `json:"view"`         // ext.<name>.<view>
    Workspace   workspace.ID   `json:"workspace"`
    Client      ClientID       `json:"client"` // proven by attachment token, never caller-asserted
    Args        map[string]any `json:"args,omitempty"`
}
type ViewEvent struct {
    ViewSession  string        `json:"view_session"`
    Handler      string        `json:"handler"`        // stable opaque handler id from the current frame
    Args         []any         `json:"args,omitempty"` // e.g. [text, eventCount] for search
    Revision     string        `json:"revision"`       // frame revision the event was produced against
    Seq          int64         `json:"seq"`
    Generation   uint64        `json:"generation"`     // daemon-assigned causal generation (SI-19)
    AckEffects   []string      `json:"ack_effects,omitempty"`   // at-most-once fence (SI-21)
    EffectResult *EffectResult `json:"effect_result,omitempty"` // e.g. pick_files → {EffectID, Payload}
}
type EffectResult struct {
    EffectID string          `json:"effect_id"`
    Payload  json.RawMessage `json:"payload,omitempty"`
}
type ViewFrame struct {
    ViewSession string       `json:"view_session"`
    Revision    string       `json:"revision"`
    InReplyTo   int64        `json:"in_reply_to,omitempty"` // event seq answered; 0 = program push
    Generation  uint64       `json:"generation"`            // causal generation of the causing admission;
    // pushes (InReplyTo=0) carry the SDK's current generation — superseded/canceled generations
    // are rejected even on the push path (SI-19); background pushes read ctx.CurrentGeneration().
    Payload     *ViewPayload `json:"payload,omitempty"`     // full frame (open/reset)
    Patch       *ViewPatch   `json:"patch,omitempty"`       // validated exactly-one-of Payload|Patch
    Effects     []Effect     `json:"effects,omitempty"`     // closed union; at-most-once by id (SI-21)
    Handlers    []string     `json:"handlers"`              // live ids (rest quarantined 2 frames)
}
// view/open: ViewOpenRequest -> ViewFrame · view/event: ViewEvent -> ViewFrame|ack
// view/close: {ViewSession, Reason} -> ack · host-API push: view/patch(ViewFrame)
```

```go
// internal/extension/cmd_palette_manifest.go (owned by the extension package)
type CmdPaletteConfig struct {
    Commands []CmdPaletteCommand `json:"commands"`
    Views    []CmdPaletteView    `json:"views"`
}
type CmdPaletteCommand struct {
    ID, Title, Section, Icon string
    Keywords        []string
    Arguments       []CmdPaletteArgument
    Action          CmdPaletteAction // tool | view | navigate | url (no client_op)
    Destructive     bool
    Confirmation    *CmdPaletteConfirmation
    DefaultShortcut string                     // daemon chord grammar; bind-only-if-free
    Execution       *CmdPaletteExecutionPolicy // optional {SingleFlight, RetrySafe *bool};
    // validated; nil → safe defaults by action kind; projected on catalog/inspect (round-3 B-007)
}
```

### Data Models

New globaldb fragment `internal/store/globaldb/schema/definitions/44_cmd_palette.sql` (Goose version `00070`, P2 — after the P1 approval migration `00069`; append-only):

| Table · column | Purpose |
| --- | --- |
| `cmd_palette_usage` — `workspace_id TEXT NOT NULL` | Scope; FK-style trigger cleanup on workspace delete (pattern: `33_extensions.sql`) |
| · `command_id TEXT NOT NULL` | Stable command id; tolerant-read pruned when unknown |
| · `use_count INTEGER NOT NULL DEFAULT 0` | Log-compressed in scoring, raw here for auditability |
| · `frecency_weight REAL NOT NULL` | Pre-decayed accumulator; decay applied at read with injected clock |
| · `last_used_at INTEGER NOT NULL` | Recents ordering + decay anchor (unix ms) |
| · `updated_at INTEGER NOT NULL` | Upsert bookkeeping. PK `(workspace_id, command_id)` |
| `cmd_palette_query_hits` — `workspace_id`, `query TEXT NOT NULL` (normalized), `command_id`, `weight REAL`, `last_used_at`, PK `(workspace_id, query, command_id)` | Adaptive query learning; prefix-matched at read; 14-day half-life; never stores argument values |
| `cmd_palette_pins` — `workspace_id`, `command_id`, `pinned_at`, PK `(workspace_id, command_id)` | Pin membership + rest-state order |

The async approval contract (round-2 B-002, deepened round 4) gets its own **tools-owned** durable model — fragment `internal/store/globaldb/schema/definitions/43_tool_approval_pending.sql` (beside `42_tool_approval_grants.sql`), migration **`00069` — landing in P1, before the public invoke surface**; the palette personalization fragment becomes `44_cmd_palette.sql`, migration **`00070` (P2)**:

| `tool_approval_pending` column | Purpose |
| --- | --- |
| `approval_id TEXT PRIMARY KEY` | The stable public id the 202 returns |
| `workspace_id TEXT NOT NULL` | Scope + delete trigger (pattern: 33_extensions) |
| `invocation_id TEXT NOT NULL UNIQUE` | End-to-end correlation AND the provider **idempotency key** |
| `target_kind TEXT NOT NULL` · `tool_id TEXT` · `target_json TEXT` | What resumes: `tool` (tool_id) or generic `client_op\|navigate\|view` target — destructive non-tool commands map to approval too |
| `command_id TEXT` | Palette command when palette-initiated |
| `args_json TEXT NOT NULL` | Args snapshot for resume — **secret-safe**: bound-secret/vault references stored as handles; commands with password-typed arguments are rejected at invoke with `cannot_defer_secrets` (a secret is never durably persisted) |
| `approval_status TEXT NOT NULL` | `pending → approved \| denied \| timeout \| canceled` (terminal, final) |
| `execution_status TEXT` | `NULL → dispatching → completed \| failed \| uncertain` — approval and execution are **separate** state machines |
| `result_json / error_json TEXT` | Terminal result/error visibility for status surfaces |
| `requested_at/expires_at/resolved_at/executed_at INTEGER` | Lifetime + boot recovery |
| `resume_fence INTEGER NOT NULL DEFAULT 0` | Claimed atomically by the `approved → dispatching` transition |

Exactly-once semantics, honestly stated: the `approved → dispatching` transition claims `resume_fence` atomically (UPDATE … WHERE approval_status='approved' AND resume_fence=0); the provider is invoked with `invocation_id` as its idempotency key; a crash between dispatch and completion recovers to `execution_status='uncertain'` — surfaced through the status contract, **never silently retried**. The single-flight lease releases on the **execution** terminal (completed/failed/uncertain-surfaced), not the approval terminal (SI-1). Boot recovery re-arms expiry for `pending` rows (expired → timeout) and sweeps `dispatching` rows to `uncertain`. Public contract (frozen in `_dx.md`): `GET /api/tools/approvals/{id}` (status + result/error) and `POST /api/tools/approvals/{id}/cancel`, with CLI/native parity — tools-owned, generic, not palette-specific. Side-table-vs-JSON: all statuses/ids/timestamps are matchable columns; only args/result/error snapshots are opaque JSON.

Side-table-vs-JSON: all three entities are matchable/rankable state → side tables with real columns; **no JSON blobs**. Config-backed (TOML, low-write): `[cmd_palette] fallback_targets []string` (default `["agent"]`), `personalization bool` (default `true`), `[cmd_palette.aliases] map[command_id]alias`; keymap overrides stay in `[window_manager.shortcuts]` (gains open id space + `palette.summon.global = ["meta+shift+Space"]`). Recents derive from `cmd_palette_usage.last_used_at` — no fourth table.

Wire types (`internal/api/contract/cmd_palette.go`): `CmdPaletteCommandsResponse{Commands, Sources, CatalogRevision}`, `CmdPaletteInvokeRequest{Workspace, Args, Client}`, `CmdPaletteInvokeResult{Status, Result, ApprovalID}`, `CmdPalettePersonalizationResponse`, `CmdPaletteViewEnvelope`, `CmdPaletteViewPatch` — shapes exactly as `_dx.md` shows them.

### API Endpoints

Handlers in `internal/api/core/cmd_palette.go` (+ `cmd_palette_views.go`, `cmd_palette_stream.go`), registered on **both** `httpapi/routes.go` and `udsapi/routes.go` (transport-parity test extended); OpenAPI ops in `internal/api/spec/registry_cmd_palette.go`; `make codegen` co-ships TS types.

- `GET /api/cmd-palette/commands?workspace=&client=` — assemble catalog (providers → resolve availability + bindings + aliases + source statuses) + personalization snapshot revision; `client` (optional) resolves client-context predicates against that attachment's snapshot; without it such commands report `requires an attached shell` — **listing never auto-selects a client**, regardless of attachment count (auto-selection exists only on invoke with exactly one attachment); 200 always (empty allowed).
- `GET /api/cmd-palette/clients` — enumerate attached clients `{client_id, kind: shell|browser, workspace, attached_at}`; the targeting source of truth for invoke.
- `POST /api/cmd-palette/commands/{id}/invoke` — validate args against declared schema (422 w/ fields) → client targeting (`client` optional: one attached → auto; many → 409 `multiple_clients` listing ids; none for client-context → 412 `no_attached_shell`) → availability re-evaluated against the targeted client's snapshot (412 + the same reason listing showed) → single-flight guard (409) → route: `client_op` forwards over the targeted client's channel; `tool` flows through `internal/tools` policy (202 `approval_pending` when gated); `navigate`/`url`/`view` are client-routed. Records usage on success.
- `POST /api/cmd-palette/usage` — client-executed usage report `{command_id, query}` (internal projection surface; same policy: normalized query only).
- `GET|DELETE /api/cmd-palette/personalization?workspace=` — aggregate summary / reset per `_dx.md`.
- `GET /api/cmd-palette/rank-signals?workspace=` — the scorer's authorized projection (frozen in `_dx.md`): `{weights{version,…}, usage[{command_id, weight, last_used_at}], query_hits[{query, command_id, weight}], pins[], revision}`; authenticated + workspace-scoped; session-memory only in the client (excluded from the IndexedDB catalog record).
- `PUT|DELETE /api/cmd-palette/pins/{command_id}?workspace=` — idempotent pin/unpin (the action panel's Pin meta-action and the CLI both call this); emits `cmd_palette.catalog.changed` for live convergence.
- `GET /api/cmd-palette/stream` — SSE via `PrepareSSE`; event `cmd_palette.catalog.changed{workspace, catalog_revision}`; no per-event replay cursor — clients reconcile on `open` by refetching (session-catalog-stream pattern); web consumes only through `createStreamEventSource`.
- `GET /api/cmd-palette/views/{id}?workspace=` + `GET /api/cmd-palette/views/{id}/stream` — extension view envelope + revision-fenced JSON Patch stream; `after` requires `stream_epoch` (extensions-dev-logs guard) with full-payload resync on fence mismatch. Programmable views add `POST /api/cmd-palette/views/{id}/open` — binds the caller's authenticated windowmanager client identity + canonical workspace, mints the session, returns `{view_session, first_frame, stream_token}` — and `POST /api/cmd-palette/view-sessions/{session}/events` (forwards UI events as `view/event`; 202 on ack-only; also carries `ack_effects: [effect_id]`). Frames arrive on `GET /api/cmd-palette/view-sessions/{session}/stream` — **session-scoped and token-authorized**: workspace/client/session ownership is enforced server-side on every stream/events/close call, and frames for other sessions are never written to the socket (isolation is server-side, never client-side filtering). Sessions auto-close on client disconnect or palette teardown, or explicitly via `DELETE /api/cmd-palette/view-sessions/{session}`.
- Settings: `GET|PATCH /api/settings/cmd-palette` (new `SectionCmdPalette`: fallback targets, personalization flag) and the extended `GET|PATCH /api/settings/window-manager` (open-id shortcuts + aliases + extension-default statuses) — both via the section machinery (`section_config_update.go`, lifecycle `Live`).

## Integration Points

- **Electron shell** (`desktop/`): new preload (`product-preload.ts` + `product-contract.ts`, modeled on the boot pair; new esbuild entry in `scripts/build-main.ts`, new `files:` entry in `electron-builder.yml`). Bridge surface: `window.compozyShell = { platform, on(event, cb), globalShortcuts: { sync(bindings), status() } }`. Web pushes the effective global-chord set (from the daemon keymap) over `sync`; main converts chords → Electron accelerators (`meta+shift+Space` → `CommandOrControl+Shift+Space`, tested converter), registers via `globalShortcut`, returns per-id `{registered|failed_in_use|failed_permission}`, and **restores the previous binding on failure** (supercmd pattern). Summon fires `shell:summon` → web opens the palette overlay; `ProductWindow.focus()` restores the window. Shell detection = bridge presence. macOS accessibility detection surfaces once at bridge init.
- **Tool policy/approvals** (`internal/tools`): tool-action invokes reuse registry/policy/approval unchanged; the palette adds no second gate.
- **Global-hotkey binding model (daemon-owned intended state + shell-reported registration status)**: intended global bindings live in `[window_manager.global_shortcuts]` (command_id → chord; **any** registry command may appear; `palette.summon.global` is just its default entry) — mutated via the settings PATCH and `compozy cmd-palette bind <id> <chord> --global` / `unbind <id> --global`, with the same atomic conflict semantics as in-app chords. The shell reconciles: it receives the effective global set over the client channel, registers via `globalShortcut`, and acknowledges per-chord `{registered | failed_in_use | failed_permission | unsupported}`; the daemon stores that ephemeral registration status per shell client and projects **both** intended and registration state on the catalog and settings GET (SD-007: a chord renders active only when confirmed registered; failure restores the previous working binding and reports it). Browser mode: the section is absent from runtime state, settings render the requires-desktop-shell reason.
- **Windowmanager client registry + fenced WebSocket** (`internal/windowmanager/clients.go`, `internal/api/core/window_manager_ws.go`): the attachment identity, context-snapshot transport, and correlated `client_command` delivery channel the palette extends — no parallel channel or registry is created.
- **Session landing**: entity results and Sessions view both call the shared attention-jump path (BR-20 single-path — root `jumpToSession` divergence is deleted).
- **SSE facade** (`web/src/lib/ticketed-event-source.ts`): all new streams consumed through it (single-use gateway tickets).

## Impact Analysis

| Component | Impact | Description and risk | Required action |
| --- | --- | --- | --- |
| `internal/cmdpalette` (+ corecmds) | new | Registry domain; medium risk (load-bearing contract) | Build with headless test suite from day one |
| `internal/windowmanager/shortcuts.go` + `defaults.go` | modified | Open bindable-id space, ext tier, summon default; collision semantics preserved | Extend canonical shortcut tests |
| `internal/extension` (manifest, validation, projection) | modified | New `cmd_palette` family; low risk (mirrors CLI-commands pattern) | Manifest fixtures + validate errors |
| `internal/api` (core/spec/contract/routes) | modified | New route family + settings section; parity + codegen enforced | OpenAPI co-ship, parity test rows |
| globaldb schema | modified | Fragments `43_tool_approval_pending.sql` (migration `00069`, P1) + `44_cmd_palette.sql` (migration `00070`, P2) | `make codegen` + migrate-suite extension |
| `web/src/systems/os` palette tree | rewritten | Registry projection replaces hardcoded groups; highest breadth | Phased with Storybook + Playwright coverage |
| Menubar menus (6 files) | modified | Items become projections; chrome untouched | Per-menu diff against registry ids |
| Settings shortcut table + new Palette section | modified/new | Whole-registry table, aliases, global section | RTL + e2e coverage |
| `desktop/` shell | modified | First product-window IPC + globalShortcut module | `_electron` e2e for summon/failure states |
| CLI + native tools | new | `cmd-palette` verb group; 2 new tool ids | `native-tool-catalog.json` regen |
| `sdk/typescript` | modified | `view-provider.ts` + view-session registry + `AbortSignal` in TransportHandler | Own vitest + conformance + a daemon-side TS fixture |
| `sdk/react` (`@compozy/extension-react`) | new | Palette React kit + reconciler; load-bearing new package | Workspace/Turbo membership; own vitest; typecheck lane guards typing drift |
| `internal/subprocess` + `sdk/go` transport | modified | Cancel frames, per-request contexts, negotiated view timeout — protocol additions shared by all provides | Transport unit + conformance suites |
| `internal/extensionprotocol` | modified | `view.provider` capability entry (+ codegen to both SDK tables) | `make codegen` + handshake tests |
| `internal/tools` approval contract | modified | Async approval primitive (`BeginApproval`/`ApprovalID`/`Resolve`-once, detached bounded lifetime) — the substrate the frozen 202 requires; palette consumes, never duplicates | Approval lifecycle suites + IT-010 matrix; approval surfaces co-ship |
| `internal/windowmanager` clients | modified | `ClientView` gains kind + volatile palette-context fields; correlated `client_command` envelopes on the existing WS | Client-registry + WS suites; IT-031 |

**Delete targets (hard cuts, no shims, no fallbacks):**

- `web/src/systems/os/components/os-command-palette-results.tsx`, `os-command-palette-window-actions.tsx`, `os-command-palette-shell-actions.tsx`, `os-command-palette-views.tsx` — replaced by `PaletteResults` projection.
- `web/src/systems/os/hooks/use-os-command-palette.ts` (god view-model) — replaced by registry hooks.
- `web/src/systems/os/lib/window-manager-command-registry.ts` (TS metadata mirror) — metadata now daemon-served.
- `web/src/systems/os/lib/window-manager-action-dispatch.ts` (if-chain) — replaced by the dispatch seam.
- `web/src/systems/os/lib/palette-view-registry.ts` closed `PaletteViewId` union + `PALETTE_VIEW_FRAMES` closed map — replaced by registry-driven view resolution.
- Closed-set shortcut validation in `internal/config/window_manager.go` (registry supplies bindable ids).
- Hardcoded menubar item lists in the six menu files (projections replace them).
- Root `jumpToSession` direct `coordinator.userOpen` landing path (BR-20).
- The declared-but-never-called `execute_hook` daemon-request seam in the extension protocol (ADR-008 — `view/event` is the interactive path; no dual seams).

## Extensibility Integration Plan

- Manifest: `resources.cmd_palette` family (commands/views with `source` | `program` tiers) in `internal/extension/manifest.go` + dedicated validation file (incl. `program: true` requires a TypeScript extension this release); SDK types in `sdk/go/types.go` + `sdk/typescript`; docs `packages/site/content/docs/extensions/cmd-palette.mdx` (new) + `manifest.mdx` (updated).
- **`view.provider` capability**: one entry in `internal/extensionprotocol/capabilities.go` (+ constants) → `make codegen` propagates to both SDK contract tables, conformance fixtures, and all three validation points; daemon service caller `internal/extension/view_source.go` via the standard gate; `DescribePayload` gains a static view-metadata field (**daemon lands first** — strict describe decoding); initialize response advertises view ids (`watch_source_kinds` pattern).
- **TS work**: `sdk/typescript` gains `view-provider.ts` registrar (inherits `ExtensionProvideSurfaces` rollback), the view-session registry class, and `AbortSignal` in `TransportHandler`; the **new `@compozy/extension-react` package** (`sdk/react/`, root-workspace + Turbo member) carries the component kit + persistent-mode reconciler + hooks + starvation guard, consuming generated vocabulary typings through its SDK dependency (drift caught by `codegen-check` + Bun typecheck); scaffold template `view-provider-ts`; and a **TypeScript fixture extension in `internal/extension/testdata/`** (closing the all-Go integration blind spot). Go SDK: untouched for views this cycle (ADR-009).
- **Subprocess protocol additions** (both peers + Go SDK where shared): cancel frames + per-request contexts, negotiated `default_view_timeout_ms`, view-class rate limiter, per-extension in-flight caps. The dead `execute_hook` daemon-request seam is **deleted** (greenfield rule; ADR-008).
- Projection: `Manager.CmdPalette(workspaceID)` — storage-free, health-gated, dev-overlay aware; **not** a ResourceKind (no codec/publish/reconcile — presentation metadata like CLI commands; checked surfaces: `surfaces/registry.go` untouched, `extensions.resources.allowed_kinds` unaffected).
- Default shortcuts: manifest field → keymap ext tier; conflicts dormant + surfaced.
- Dev mode: existing watch/reload emits `cmd_palette.catalog.changed`; palette contributions hot-reload (US-031); program reload drops view sessions with the "view reloaded" note.
- Hooks: **no new hook families this cycle** — tool-backed executions already traverse tool hooks; palette-specific events (`command.pre_invoke` etc.) are deliberately deferred (checked: `internal/hooks/events_catalog.go` untouched).
- MCP sidecars/bridge SDKs: unaffected (no protocol change; checked `internal/network`, bridge contract surfaces).

## Agent Manageability Plan

Exactly as `_dx.md` freezes: `compozy cmd-palette list|inspect|invoke|clients|personalization|pin|unpin` plus the mutation verbs `bind|unbind|alias set|alias clear|bindings` (atomic through the settings PATCH path, `--overwrite` semantics, structured errors identical to HTTP — closing the US-034 mutation parity), `/api/cmd-palette/*` on HTTP+UDS, native tools `compozy__cmd_palette_list` + `compozy__cmd_palette_invoke` (descriptors, risk pass-through, catalog snapshot regen), scalar settings through `compozy config set` (`tool_surface.go` registration; aliases/bindings mutate via the verbs, not scalar paths). Deterministic error table in `_dx.md` § Errors; reasons byte-identical across UI and structured errors (BR-8).

## Config Lifecycle

- New `internal/config/cmd_palette.go`: `CmdPaletteConfig{FallbackTargets, Personalization, Aliases}` + defaults + `Validate()` (alias grammar, known fallback targets).
- `tool_surface.go`: register `cmd_palette.fallback_targets`, `cmd_palette.personalization` (aliases managed via settings section, not scalar surface).
- Settings: new `SectionCmdPalette` (build/diff/apply per `shell_section.go` template) + extended `window_manager_section.go` (aliases + open-id shortcuts diff paths); lifecycle class `Live` (`lifecycle.go` root `cmd-palette`).
- `[window_manager.shortcuts]` validation now accepts registry ids incl. `ext.*` and the new `palette.summon.global` default in `internal/windowmanager/defaults.go`.
- Docs: generated config reference regen (`make codegen`); `packages/site` configuration/shortcuts page updated (palette bindings + summon + aliases).
- Settings-affected check: no existing key removed; `[window_manager]` semantics extended, never broken.

## Testing Approach

- **Unit (Go)**: cmdpalette descriptor validation, catalog assembly/revision determinism, personalization signal maintenance (decay math, pruning, weight-version serving — **scoring itself lives only in the TS suite**, ADR-003 amendment), view vocabulary validate/fallback, view-service admission, approval-coordinator lifecycle, manifest validation errors, keymap ext-tier/collision, config validation. Clock injected everywhere; store faked at the sqlc boundary only.
- **Unit (web)**: projection hooks, context evaluator, dispatch seam, filter/blend (cmdk `shouldFilter={false}` path), args/confirmation state machines — extending the canonical suite `use-os-command-palette.test.tsx` lineage and the pure-lib suites; new `packages/ui` command tests/stories for the custom-filter path.
- **Integration (Go, `+integration`)**: `/api/cmd-palette/*` handlers with real globaldb (migration applied), transport parity rows, settings sections, SSE stream fencing, extension fixture manifest → projection → catalog.
- **E2E runtime (Go harness)**: agent journey — list → inspect → invoke (approval path, no-shell path) via UDS.
- **E2E web (Playwright)**: operator journeys from `_uiux.md` surfaces (root search, views, panel, args, confirmation, rebind, fallback) extending `os-shell.spec.ts`.
- **E2E desktop (`_electron`)**: summon, global-hotkey registration failure surface, browser-mode gating — in `desktop/e2e/_electron/__tests__/`.
- Contract co-ship: OpenAPI + generated TS + `native-tool-catalog.json` + E2E mock matchers ship with the contract change (`eng-contract-codegen-coship`).
- **View-program level**: `sdk/react` vitest owns the kit (reconciler → frames, handler-id stability, patch generation, starvation warning); `sdk/typescript` vitest owns the view-provider registrar/session registry; daemon integration runs a **TypeScript fixture extension** with a program view end-to-end (open session → events → patches → crash/degraded paths); event-loop property tests pin event-counter discard and quarantine semantics.
- Env deps: integration needs migrated globaldb temp store; desktop E2E needs built shell (`make desktop-build`); extension tests need TWO fixture extensions — a Go one with a `cmd_palette` block (commands/shortcuts/declarative view) and a TypeScript one with a program view (the missing in-repo exemplars).

Every concrete case lives in [`_tests.md`](_tests.md).

## Development Sequencing

### Build Order

Every slice ships its **complete public surface** — both transports, OpenAPI + generated TS, CLI, native tools where applicable, `skills/compozy/` updates, owning docs pages, and tests — in the same slice; internal-only work stays private until its slice co-ships (no partial-surface task boundaries).

0. **P0 Design pass** — the ten `docs/design/opendesign/command-palette/` artboards (inventory in File References § Design and Analysis Sources), produced by the operator **before execution, outside the task graph**; every UI-bearing task names its artboards as the visual contract (screenshot bundles in the owning verification tasks) and blocks — never improvises — if an artboard is missing at execution time.
1. **P1 Registry vertical slice** — `internal/cmdpalette` + corecmds; catalog/invoke/clients API on HTTP+UDS + OpenAPI/TS + SSE; CLI `list|inspect|invoke|clients`; native tools + catalog regen; `skills/compozy/` + owning docs page; web hydration + dispatch seam + full core absorption; menubar/cheatsheet/settings-table derivation; legacy deletes. Gate: `make gate` + transport parity + contract co-ship (codegen-check) + shortcut suites.
2. **P2 Personalization slice** — migration `00070`, scoring port + snapshot, usage recording, pins/recents/ghost, empty-query surface, `personalization` CLI + docs. Gate: ranking property tests + migrate suites.
3. **P3 Execution UX** — action panel, inline args, confirmation, feedback lifecycle, single-flight UI (artboard-bound; no new public contract). Gate: web unit + Playwright rows.
4. **P4 Keyboard slice** — open-id keymap + aliases storage, `bind|unbind|alias|bindings|pin|unpin` CLI verbs, settings shortcut table + Palette section (personalization controls only — the fallback toggle ships with its behavior in P8), cheatsheet extension, config keys + docs. Gate: settings/keymap suites + E2E-016/025. **← MVP line**
5. **P5 View system slice** — vocabulary structs + patch engine + generated TS, shell generalization, all-domain built-in views, Detail/Form/Grid, view routes on both transports + docs. Gate: vocabulary round-trip fixtures + Playwright view rows.
6. **P6 Extension contribution slice** — manifest family (both tiers) + validation + projection + default-shortcut tier + dev reload + settings panels + Go fixture + extension docs; **P6b view-program runtime**: `view.provider` capability + `view_source.go` + session registry + `view/patch` SSE fan-out + subprocess protocol additions (cancel/timeout/rate/caps) + TS SDK (`view-provider.ts`, AbortSignal) + `@compozy/extension-react` package + TS fixture + `execute_hook` deletion + SDK/react docs. Gate: Go+TS fixture integration suites + conformance.
7. **P7 Desktop global hotkeys slice** — preload bridge, shortcuts module, global-binding model reconciliation, summon, failure surfaces, settings section, `_electron` E2E; owning docs page: **`packages/site/content/docs/desktop/global-hotkeys.mdx` (new)** — shell-only registration, macOS Accessibility, browser behavior, failure states (+ `configuration/shortcuts.mdx` cross-link).
8. **P8 Fallback + QA close** — NL fallback + fallback settings, QA-debt walk (`ET-palette-sessions-view-switch`) + new scenario walks, integration polish. Gate: `make gate-full` at close.

### Technical Dependencies

P2+ depend on P1 (registry contract). P6 depends on P5 (view vocabulary) and P1. P7 depends only on P1 (chord source) — parallelizable after it. Migration lands at P2 start. No external-service dependencies; Electron shell already in-repo.

## Monitoring and Observability

- `slog` structured fields on invoke: `command_id`, `source`, `workspace_id`, `exec_site` (derived from the action kind), `client_id`, `outcome`, `duration_ms`, `catalog_revision`.
- **Canonical event matrix** (registered in `internal/events/names.go`, family `cmd_palette`; low-frequency domain operations only — keystrokes/patches stay metrics): `catalog.changed{workspace, revision}` · `command.invoked{workspace, command_id, source, exec_site, outcome, duration_ms, invocation_id, approval_id?}` (terminal outcomes only; correlates the 202 lifecycle end-to-end) · `personalization.reset{workspace}` · `pin.changed{workspace, command_id, pinned}` · `binding.changed{workspace, command_id}` · `alias.changed{workspace, command_id}` · `view_session.opened|closed|degraded|circuit_broken{workspace, view, extension, client, view_session}` · `global_hotkey.registration_failed{workspace, client_id, command_id, chord, reason}` (bridge-status-reported; fields distinguish per-client registrations). Effect delivery failures log with `effect_id`. Web consumption uses named SSE listeners (L-017; UT-104).
- Counters/logs: invocation failures by error code, ranking-store degraded (tolerant-read fallback engaged), view-payload validation failures per extension (capability-gap telemetry per ADR-007), event-admission rejections per session, global-hotkey registration failures (shell log + bridge status).

## Technical Considerations

### Key Decisions

- **Aliases live in `[cmd_palette.aliases]` but ride the window-manager settings PATCH** (frozen `_dx.md` surface): the section apply writes the cmd_palette root — sections are presentation, roots are storage; documented in the section code.
- **Entity search stays client-side** over palette-open-gated domain queries (shipped sessions pattern generalized; legacy-commander cache-search precedent) — no new daemon search API; views own deeper filtering.
- **Built-in views render natively** through the shared normalized view model; only extension views cross the wire as payloads — one renderer, two sources.
- **Invocation lifetime belongs to the tools layer — which this spec extends with an asynchronous approval contract.** Today's `ApprovalBridge.RequestToolApproval` is synchronous and context-bound (no id, no resume); the palette's frozen 202 surface therefore requires extending the canonical approval owner in `internal/tools` — never a palette-local pending table: `BeginApproval → {ApprovalID}` (stable, public), a detached bounded lifetime, and `Resolve(approved|denied|timeout|canceled)` that resumes provider execution **exactly once**. The single-flight lease is held by that primitive until the terminal outcome; its approval surfaces (API/CLI/UDS/native) co-ship in the same slice; daemon restart clears in-memory guards while pending approvals resolve through the tools layer's durable state. The palette consumes this contract and owns nothing of it.
- **Usage recording is fire-and-forget** (`POST /usage` for client-executed; inline for daemon-executed); failures log, never block.
- **Catalog caching**: web persists the last-known **structural** catalog in an IndexedDB record `cmd_palette.catalog` keyed by `(canonical workspace id, contract version)` — descriptors/sources/bindings/aliases only, never client-resolved availability and never rank signals; on hydration, availability is re-evaluated against the current authenticated client's context, so one tab can never cold-hydrate another's resolved state; one bounded record per workspace, corrupt or version-mismatched entries dropped with a full refetch — instant open, honest staleness (BR-19).
- **Virtualization**: `@tanstack/react-virtual` (already in-repo) for domain views/grids beyond the mount cap; root stays capped with exact overflow notes (Open Q2 resolved: caps at root, virtualization in views).
- **Weights**: every ranking constant in `WeightsV1` (one file, versioned) — supercmd magic-number lesson.
- **Workspace resolution is the shared security boundary (L-033)**: public `workspace` inputs (CLI `--workspace`, HTTP `?workspace=`, native-tool params) accept ID/name/path and resolve through the canonical resolver precedence (`internal/cli/workspace_resolution.go`; cwd-derived default; the flag is never required); native tools bind the trusted session workspace via the dispatch input binding **before** validation, so callers cannot spoof scope; every daemon interface, storage row, SSE payload, and cache key carries the typed canonical workspace ID only; foreign non-operator references are rejected centrally — no palette-specific ref grammar exists.
- **Attached clients: the windowmanager client registry is the single authority — no second registry.** The palette reuses `internal/windowmanager`'s existing workspace-partitioned client identity (`RegisterClient`/`Clients`/`ClientView`, already registered over HTTP/UDS/CLI with the fenced WebSocket stream); this spec extends `ClientView` with `kind` (shell|browser) and the volatile palette-context fields (focused window, presentation, window counts, destination intent), refreshed as a debounced snapshot over the existing stream. `GET /api/cmd-palette/clients` is a read **projection** of that registry. **Two authorization paths, both explicit**: (1) *self-originated* client-bound calls (the web client opening its own view sessions, invoking against itself) prove attachment with the **attachment token** minted at registration (`X-Compozy-Client-Token`; daemon validates token ↔ ClientID — a bare caller-asserted id is never trusted); (2) *control-plane* calls (CLI over operator UDS, privileged HTTP, native tools under session identity) target a validated `windowmanager.ClientID` under their **own** authorization — they select a client, never impersonate one, and carry no attachment token. Spoofed tokens and foreign-identity targeting are rejected with structured errors. One predicate evaluator — daemon-owned, definition shared with the web — resolves client-context availability against a named client's snapshot; the web evaluates the same predicates locally for latency, and invoke re-evaluates daemon-side against the client it targets. **Delivery**: `client_op` actions route through the existing windowmanager manager-command primitive where one exists; remaining palette-only UI operations (open overlay, navigate) ride the same fenced WebSocket as correlated `client_command` envelopes `{command_id, op, payload}` with acknowledgement/result frames, a bounded delivery timeout, and disconnect semantics (undelivered → structured `client_disconnected` failure — never silent). Targeting: exactly one attached client → auto-selected; multiple → explicit `client` required (`multiple_clients` listing ids — available on CLI `--client`, HTTP `client`, and the native-tool `client` param alike); zero → `no_attached_shell`. Listed without a client, client-context commands resolve `available:false, reason:"requires an attached shell"`.
- **React-kit packaging**: the kit is a **separate workspace package `@compozy/extension-react`** at `sdk/react/` (user decision) — the SDK core stays React-free by construction, and `react`/`react-reconciler` are ordinary dependencies of the kit only. In-monorepo everything versions together, so pairing cost is nil; the binding requirement is **typing freshness**: the kit consumes the generated frame-vocabulary typings (via its `@compozy/extension-sdk` workspace dependency), `make codegen` regenerates them from the Go source of truth, and `codegen-check` + the Bun typecheck lane are the drift gates — a Go vocabulary change that breaks the kit fails the gate, never ships stale. New root-workspace + Turbo membership for `sdk/react` (build/typecheck/test lanes). The full component/hook inventory is frozen in `_dx.md` § "The React kit at a glance"; kit additions follow the ADR-004 vocabulary rule.

### Known Risks

- cmdk 1.1.1 with `shouldFilter={false}` + external ordering is under-tested in-repo — new packages/ui tests/stories are mandatory before the projection lands (arch risk 9).
- Catalog scale DOM cost at root — mitigated by group caps + section assembly; measured in Storybook AtScale stories.
- `globalShortcut` silent failures and macOS accessibility — mitigated by typed status + restore-previous + settings surface; non-QWERTY Electron limitation documented in recorder copy.
- Patch-stream ordering/fencing bugs — mitigated by revision fences + full resync + property tests on patch application.
- Absorption breadth (every surface touched) — mitigated by P1 gating and per-surface parity checks against the legacy inventory before deletes.
- "desktop"/"window" homonyms (web WM vs Electron shell) — glossary + code comments disambiguate at every seam.

## Safety Invariants

1. A non-idempotent command id has at most one in-flight invocation per workspace; the daemon rejects the second with `already_running` (409). For approval-gated invocations the guard is owned by the tool execution/approval primitive and held across the detached lifetime — released only on a terminal approval outcome (approved-and-completed, denied, timeout, canceled), never at the 202 and never leaked past a terminal state; daemon restart clears the in-memory guard (the tools layer owns the durable state).
2. The web applies a `ViewPatch` only when `From` equals its current view revision; any gap triggers full-payload resync — patches are never applied out of order.
3. Catalog renders are revision-consistent: a hydration pass either applies a complete catalog or keeps the previous one; no partial merges.
4. Keymap, alias, and palette-config mutations flow exclusively through the settings OverlayEditor path; nothing else writes `config.toml`.
5. Catalog membership changes only on enable/disable/remove; health transitions change availability and source status, never membership. Either transition atomically produces a new catalog revision covering commands, views, default bindings, and source statuses together; an unhealthy source serves its last-known validated descriptors flagged `available:false` with its status/reason, and recovery restores availability with a new revision.
6. Personalization tables never store argument values, password-field content, or vault data; query strings are recorded normalized and only from the pre-selection query (US-015.EC-4, US-019.EC-3).
7. Every personalization row carries `workspace_id`; workspace deletion cascades via trigger; every read filters by `workspace_id` — no cross-workspace read path exists.
8. `client_op` commands require an attached shell client; the daemon rejects (`no_attached_shell`) rather than queueing or pretending.
9. Destructive/approval-gated executions traverse tool policy on every path — UI confirmation never substitutes for policy approval on agent invocations, and agent approval never skips the declared UI confirmation semantics.
10. A chord is displayed as active only when its registration is confirmed (daemon keymap for in-app; bridge status for OS-global); failed global registration restores the previous binding.
11. Core command ids are validated unique at daemon boot (duplicate = boot failure); `ext.*` collisions reject the later registration at manifest load with a diagnostic.
12. View payloads are schema-validated daemon-side before serving; unknown elements at render hit the Null-Object fallback with telemetry — never a crash, never silent omission of a `requires`-failed element whose fallback is content.
13. A view session belongs to exactly one attached client (bound at `open` from the authenticated windowmanager client identity) and one extension instance; ownership of workspace/client/session is enforced **server-side** on every stream/events/close call (frames for other sessions never reach a socket — client-side filtering is not isolation); re-entrant calls are ownership-checked against the minting extension and rejected across extensions (clarify-runtime rule); teardown is idempotent, cancels in-flight work, and fires automatically on client disconnect and palette dismiss.
14. Controlled-value events carry a monotonic event counter; the host discards any echoed value whose counter is stale — a program can never revert the operator's typing.
15. Handler ids are stable across re-renders; disposed handlers are quarantined for two frames and in-flight events on them drop silently — never an error, never a mis-fire.
16. Degraded and circuit-broken views always retain the last-good render; the palette shell (Esc, ⌫, navigation) stays responsive regardless of any program's state; a program crash can never take down the palette or another view.
17. Client-context availability has one evaluator and one identity authority: attachment identity IS the windowmanager client registry (no palette-local registry exists); the daemon resolves predicates against a named client's latest snapshot, and invoke re-evaluates against the client it targets — the availability a caller listed and the context a dispatch runs against can never diverge; with multiple attachments, targeting is always explicit or a structured error, never a silent pick; client-bound delivery is correlated (command id + ack/result) with disconnect surfacing as a structured failure.
18. View programs gain no unmediated daemon capability: a row/panel action declares **exactly one** of a policy-dispatched `Action` union (host-executed — availability/risk/trust/approval run before any extension code hears of it) or a view-local `Handler` id (opaque to the daemon: view state and the extension's own domain logic under its existing grants — granted nothing new); host effects are the closed, at-most-once union of SI-21 under the same rules as command actions; daemon-state access flows through the permission-gated Host API; destructive entries require a declared confirmation regardless of shape; Tier-1 `source` tools must be read-only risk class (validate-time rejection — declarative views load without policy prompts by construction).
19. Per view session, at most one coalescible event (search/chip/visible-range) is in flight: a newer event cancels the superseded request (SDK receives the abort) and the daemon discards frames answering a superseded sequence. Every handler admission carries a daemon-assigned **causal generation**; every frame, patch, and effect caused by that handler — including `InReplyTo=0` pushes — carries it, and output from a canceled or superseded generation is rejected regardless of revision ordering (revision proves message order; generation proves semantic freshness — a canceled handler that races its abort can never overwrite newer rows). Independent action events are capped per session; excess is rejected with a structured busy error, never queued unbounded.
20. Late view frames after session close, extension restart, or generation bump are disposed by session-id validity — never applied, never an error surfaced to the operator.
21. Host effects are **at-most-once by effect id**: each `Effect` carries a stable id, the client acknowledges execution (`ack_effects`), and resync/replay paths re-deliver frames **without** their already-acked effects — a reconnect can never repeat a clipboard write, URL/app open, toast, or file-picker prompt; `pick_files` results correlate back to the waiting handler by effect id.

## File References

### Repo Files

**Palette web (rework/delete inventory)**
- `web/src/systems/os/components/os-command-palette.tsx` — root/view branch + single mount; every rework starts here.
- `web/src/systems/os/hooks/use-os-command-palette.ts` — the god view-model being replaced; its member list is the absorption checklist.
- `web/src/systems/os/components/os-command-palette-results.tsx`, `os-command-palette-window-actions.tsx`, `os-command-palette-shell-actions.tsx`, `os-command-palette-views.tsx` — delete targets; their rows enumerate the core-command inventory to absorb.
- `web/src/systems/os/lib/palette-view-stack.ts`, `hooks/use-os-palette-view-stack.ts`, `components/os-palette-view-shell.tsx`, `os-palette-view-stack.tsx`, `os-palette-breadcrumb.tsx`, `os-palette-footer.tsx` — the shipped stack mechanics P5 generalizes (breadcrumb ≤3, selection survival).
- `web/src/systems/os/hooks/use-os-palette-sessions-view.tsx`, `lib/palette-session-filters.ts`, `components/os-palette-session-{chips,row}.tsx`, `os-palette-view-note.tsx` — the Sessions grammar every domain view generalizes; 150-row cap precedent.
- `web/src/systems/os/lib/window-manager-command-registry.ts` — TS mirror (delete); its metadata seeds core descriptors.
- `web/src/systems/os/lib/window-manager-action-dispatch.ts` + `hooks/use-os-window-commands.ts` — the dispatch if-chain to replace + the shared window model both palette and menubar consume.
- `web/src/systems/os/hooks/use-os-shortcuts.ts` — the single keydown listener; gains registry-driven ids, keeps guards.
- `web/src/systems/os/lib/window-manager-shortcuts.ts` — chord parse/label/conflict/cheatsheet derivation to reuse.
- `web/src/systems/os/lib/window-manager-config.ts:31-33` — effective-keymap merge the projection reads.
- `web/src/systems/os/hooks/use-desktop-overlays.ts`, `components/desktop-shell.tsx:315-320`, `hooks/use-desktop-shell-body.ts:159-219` — overlay slot, mount, shortcut handlers.
- `web/src/systems/os/lib/app-catalog.ts` — `OS_APP_DESCRIPTORS`; seeds app commands + navigate targets.
- `web/src/systems/os/stores/window-manager-store.ts:181-205` + `window-manager-store-types.ts` — `paletteViewStack`/`paletteIntent` transitions (kept).
- `web/src/lib/status-tone.ts`-exported dictionary + `compareAttentionFirst` + `useAttentionJump` + `routing-coordinator.ts` land-on-session — the shared authorities BR-20 and the anti-duplication rules bind.
- `web/src/systems/os/components/menubar/*.tsx` + `desktop-menubar.tsx` + `os-menubar.tsx` — six menus to project + surviving chrome.
- `web/src/systems/os/components/os-shortcuts-dialog.tsx` — cheatsheet derivation to extend.
- `web/src/systems/settings/components/layouts/window-manager-shortcut-table.tsx` + `use-window-manager-shortcut-recorder.ts` — the rebinding UI S12 extends.
- `web/src/routes/_app/settings/-extensions-settings-page.tsx` — S16 host.
- `web/src/systems/extensions/adapters/extensions-api.ts`, `lib/query-{keys,options}.ts`, `hooks/use-extensions.ts` — the adapter/hook pattern for the new cmd-palette client.
- `web/src/lib/ticketed-event-source.ts` + `web/src/systems/session/hooks/use-session-catalog-streams.ts` — mandatory SSE facade + the invalidate-on-open consumer pattern.
- `web/src/integrations/tanstack-query/reconcile-installed-extension.ts` — today's mutation-local refresh the SSE stream supersedes.
- `web/src/components/assistant-ui/session-command-menu-model.ts` + `web/src/systems/session/hooks/use-session-commands.ts` — the daemon-sourced command-menu precedent (ranking/grouping/inert-unavailable) — pattern source, not a unification target.
- `packages/ui/src/components/command.tsx` + `__tests__/command.test.tsx` + `stories/command.stories.tsx` — cmdk wrappers; the `shouldFilter={false}` path needs new tests/stories here.
- `packages/ui/src/components/sonner.tsx` + `web/src/lib/user-feedback.ts` + `web/src/systems/os/components/attention-toast.tsx` — feedback system (survives unmount) US-017 rides.
- `web/src/components/assistant-ui/hooks/use-thread-scroll-controller.ts` — the only in-repo `@tanstack/react-virtual` usage; virtualization precedent.

**Go runtime**
- `internal/windowmanager/shortcuts.go` + `defaults.go` + `shortcut_binding.go` — action ids, chord grammar, canonicalization/collision, defaults (incl. `palette.open`); the id-space this spec opens.
- `internal/command/types.go` + `catalog.go` — Lane/Descriptor/Catalog + sha256 Revision + precedence policy; the registry's closest in-repo precedent.
- `internal/extension/manifest.go`, `command.go`, `command_manifest.go`, `command_schema.go`, `manager.go`, `manager_resource_loading.go`, `dev.go` — manifest structs, live command projection (health gate), schema→flag projection, dev overlay; the contribution machinery to extend.
- `internal/extension/surfaces/registry.go` — Surface table (explicitly untouched; cmd_palette is not a ResourceKind).
- `internal/tools/registry.go`, `policy.go`, `builtin_ids.go`, `builtin/extensions.go`, `builtin/testdata/native-tool-catalog.json` — dispatch/risk/approval the invoke path reuses; native-tool declaration pattern + codegen contract.
- `internal/config/shell.go`, `tool_surface.go:157-158`, `window_manager.go`, `lifecycle/lifecycle.go:181` — config struct/validation/CLI-surface/lifecycle templates for `[cmd_palette]`.
- `internal/settings/shell_section.go`, `window_manager_section.go`, `models.go:81`, `section_config_update.go:13-42`, `config_apply_helpers.go:59` — section build/diff/apply machinery for the new + extended sections.
- `internal/api/core/settings.go:162-205`, `session_commands.go`, `session_catalog_stream.go`, `sse.go`, `extensions_dev.go:124-164`, `extensions_commands.go`, `parsers.go:67` — handler/SSE/replay-guard patterns; `httpapi/routes.go` + `udsapi/routes.go` + `internal/api/spec/registry_extensions.go` — registration + OpenAPI pattern; `transport_parity_integration_test.go` — parity enforcement.
- `internal/events/names.go` + `registry.go` — event-name registry for `cmd_palette.catalog.changed`.
- `internal/store/globaldb/schema/definitions/` (esp. `33_extensions.sql`, `42_tool_approval_grants.sql`) + `schema/migrations/` (head `00068`) + `queries/` + `sqlc.yaml` + `internal/store/migrate_streams_test.go` — schema fragment placement, delete-trigger pattern, migration process, canonical suites.
- `internal/cli/extension.go`, `extension_commands.go`, `extension_exec.go`, `client_extension_commands.go`, `app_control.go` — CLI verb/exec/client patterns for `cmd-palette` verbs; existing shell control client.

**Desktop shell**
- `desktop/src/main.ts` — entry; globalShortcut module wiring point.
- `desktop/src/window/product-window.ts` — no-preload window gaining the bridge; `before-input-event` precedent.
- `desktop/src/boot/boot-preload.ts` + `boot-contract.ts` — the contextBridge template for `product-preload.ts`/`product-contract.ts`.
- `desktop/src/window/security.ts` + `navigation-policy.ts` — default-deny posture the bridge must respect.
- `desktop/scripts/build-main.ts` + `electron-builder.yml` — esbuild entries + `files:` + fuses (new preload must be added to both).
- `desktop/src/control/control-server.ts` — token-authenticated UDS control precedent.
- `desktop/e2e/_electron/__tests__/shell.spec.ts` + `fixtures.ts` — the desktop E2E harness for summon/hotkey tests.

**QA + docs**
- `docs/qa/scenarios/ET-palette-nested-views.md` (pass), `ET-palette-sessions-view-switch.md` (**blocked-verify — resolve in P8**), `ET-web-command-palette-shortcuts.md` — canonical scenarios to extend/reset, never duplicate.
- `packages/site/content/docs/configuration/shortcuts.mdx`, `packages/site/content/docs/extensions/manifest.mdx` + `commands.mdx` — doc pages this spec updates (+ new `extensions/cmd-palette.mdx` at P6 and new `desktop/global-hotkeys.mdx` at P7).
- `skills/compozy/` — official skill needs the new CLI verbs/tool ids/routes (Impact Audit row).

### Competitor References

- `.resources/supercmd/src/renderer/src/utils/root-search-ranking.ts` — the scoring model to port (match kinds, weights, transitivity comment).
- `.resources/supercmd/src/shared/root-search-ranking-state.ts` — frecency + input-history decay/prune as pure functions; the Go port shape.
- `.resources/supercmd/src/renderer/src/utils/root-search-sections.ts` — rank→sections with promotion floors.
- `.resources/supercmd/src/renderer/src/raycast-api/action-runtime-registry.tsx` + `action-runtime-overlay.tsx` — action registry + ⌘K panel anatomy.
- `.resources/supercmd/src/renderer/src/raycast-api/list-runtime.tsx:258` — surgical virtualization pattern.
- `.resources/supercmd/src/renderer/src/hooks/useLauncherCommandModel.ts` — pure derivation pipeline + empty-query grouping.
- `.resources/supercmd/src/renderer/src/settings/HotkeyRecorder.tsx` — e.code capture rules.
- `.resources/supercmd/src/main/main.ts:~14839` — typed duplicate-conflict + restore-previous registration.
- Legacy commander (cited by path in `analysis/01_legacy_commander.md`; repo `/Users/pedronauck/Dev/compozy/compozy-code` — not vendored under `.resources/`): descriptor contract `packages/tauri/src/app-core/shared/commander.ts`, conflict logic `src-tauri/src/plugins/commander/shortcuts.rs`, formatter `renderer/lib/commander/shortcut-formatter.ts`.

### Design and Analysis Sources

- `analysis/01_legacy_commander.md` — port/skip verdicts feeding the descriptor and keymap design.
- `analysis/02_supercmd.md` — adopt/avoid verdicts feeding ranking, action panel, stack, hotkeys.
- `analysis/03_ext_views_research.md` — the json-render comparison behind ADR-007.
- `analysis/04_architecture_gaps.md` — Electron/settings/schema/SSE/menubar implementation map + 13 risks.
- `analysis/05_raycast_vicinae_runtime.md` — the programmable event-loop reference (handler ids, event counters, batching, worker lifecycle) behind ADR-008.
- `analysis/06_hosts_eventloop.md` — cross-host wire patterns (Slack/LSP/A2UI/MCP Apps) → event vocabulary, budgets, failure semantics.
- `analysis/07_supercmd_interactivity.md` — in-process vs serializable split; the MenuBarExtra out-of-process loop; the 3 bugs our port fixes.
- `analysis/08_compozy_extension_channel.md` — the provides-RPC verdict + per-language SDK seam tables (TS 12, Go 9) + protocol gap list.
- `analysis/09_react_in_go_alternatives.md` — the React-in-Go alternatives table and ranked verdict behind ADR-009; tier-composability precedent.
- `sdk/typescript/src/extension-provide-surface.ts`, `watch-source.ts`, `connectivity-provider.ts`, `transport.ts` — the registrar/kind-registry/transport patterns the view-provider TS work extends (exact anchors in analysis/08).
- Vicinae (`vicinaehq/vicinae`) — cited by concept, not vendored under `.resources/`: the C++-daemon Raycast-compatible host whose IPC vocabulary ADR-008 mirrors.
- `.compozy/tasks/_archived/herdr-parity/adrs/adr-003.md` + `_spec.md` (BR-32) + `memory/MEMORY.md` — binding palette constraints and anti-duplication rules.
- `docs/design/opendesign/herdr-parity/herdr-parity-palette-sessions.html` + `DESIGN-NOTES.md` — shipped palette visual contract; the new artboards extend its grammar.
- `docs/design/opendesign/command-palette/` (operator-produced in **P0**, before any UI-bearing slice) — ten artboards per `_uiux.md`: `command-palette-root.html`, `command-palette-root-states.html`, `command-palette-view-shell.html`, `command-palette-view-bands.html`, `command-palette-domain-list.html`, `command-palette-form-grid.html`, `command-palette-action-panel.html`, `command-palette-args-confirmation.html`, `command-palette-settings.html`, `command-palette-settings-palette.html`; each implementation task cites its artboards as the visual contract.

## Assumptions and Defaults

- Global summon default: `meta+shift+Space` (rebindable; shell-only registration).
- Fallback targets default `["agent"]`; `personalization` default `true`.
- Ranking constants v1: frecency half-life 30 days, query-learning half-life 14 days, score deadband 12, entity-section visible cap 6 (+ exact overflow note), domain-view mount cap 150 with virtualization beyond — all in `WeightsV1`, tunable in one file.
- Alias grammar: 1–32 chars, no whitespace. Icons: Lucide token set + single emoji grapheme.
- Extension default chords bind only when entirely free; enable-order breaks ties.
- Schema: approval fragment `43_tool_approval_pending.sql` → migration `00069` (P1); palette fragment `44_cmd_palette.sql` → migration `00070` (P2); globaldb stream; settings section name `cmd-palette`.
- Context-key set v1 (closed): `window.focused`, `window.floating`, `window.stacked`, `desktop.windowCount`, `scope.global`, `shell.desktop`, `session.focused.state`, `workspace.trusted`.
- View-program envelope: soft budget 150 ms (busy indicator, previous rows retained), hard ack 3 s, circuit-break at 3 consecutive misses, frame clock ~60 fps with content-hash dedupe, handler quarantine 2 frames, per-client sessions, view open budget 5 s with manifest skeleton.
- View resource caps v1 (host-enforced, named constants): ≤ 1,500 components per frame · ≤ 500 rows per patch · ≤ 256 KiB per frame/patch · 1 in-flight coalescible event + ≤ 4 concurrent independent action events per session (excess → structured `view_busy`).
- Result sets declare `complete: bool` (LSP model): `true` → host filters locally between updates; `false` → throttled round-trips.
- The composer `/` menu, deeplinks, background commands, chord sequences, and Go view programs remain out (Part I Non-Goals).

## Compozy Impact Audit

- **Native tools**: adds `compozy__cmd_palette_list` + `compozy__cmd_palette_invoke` (descriptors, schemas, risk pass-through, capability gates) → `internal/tools/builtin_ids.go` + `builtin/` + `native-tool-catalog.json` regen; existing tool IDs unchanged (checked `builtin_ids*.go`).
- **Extensibility and hooks**: new manifest family `resources.cmd_palette` (two view tiers) + live projection + `view.provider` capability (codegen to both SDK tables) + TS SDK react layer + subprocess protocol additions (cancel/timeout/rate) + `execute_hook` seam deleted + docs; hooks catalog explicitly unchanged this cycle (checked `internal/hooks/events_catalog.go`); registries: Surface table untouched (not a ResourceKind); bridge SDKs/MCP unaffected (no bridge-protocol change; checked bridge contract surfaces).
- **Workspace data isolation**: new workspace-scoped data (usage/query-hits/pins) carries `workspace_id` PK columns + delete triggers; reads always workspace-filtered; catalog/SSE payloads carry canonical workspace IDs; public refs (ID/name/path) resolve through the canonical resolver and native tools bind trusted session scope before validation (L-033); personalization snapshot never crosses workspaces (Safety Invariants 6–7); global-scope entity display carries per-row workspace labels (US-007); two-workspace isolation proven by IT-032 across CLI/HTTP/UDS/native paths.
- **Official Compozy skill**: `skills/compozy/` updates co-ship with each slice that first exposes a public contract — P1 (list/inspect/invoke/clients + native tools + routes), P2 (personalization), P4 (bind/unbind/alias/bindings/pin/unpin), P6 (extension palette family + view.provider). Glossary co-ships likewise: `cmd_palette` identifier entry (ADR-005) at P1; `view.provider` provide-surface entry at P6.

## Architecture Decision Records

- [ADR-001: One unified command registry with full OS coverage](adrs/adr-001.md) — one catalog absorbs both existing ones; every command enumerable, bindable; full-domain search.
- [ADR-002: Extensions contribute palette commands, views, and default shortcuts](adrs/adr-002.md) — all three contribution surfaces open, superseding the herdr deferral; data-only in the renderer.
- [ADR-003: Ranking is frecency plus adaptive query learning, persisted by the daemon per workspace](adrs/adr-003.md) — supercmd-style personalization as daemon state.
- [ADR-004: View vocabulary is the full Raycast kit — List, Detail, Form, Grid](adrs/adr-004.md) — supersedes herdr's list-only constraint; fixed vocabulary preserves the anti-mini-app intent.
- [ADR-005: The domain identifier is `cmd_palette`](adrs/adr-005.md) — snake/kebab mapping across manifest, config, CLI, routes, tools, events.
- [ADR-006: The command registry is daemon-canonical with a web projection](adrs/adr-006.md) — Go owns the catalog; web evaluates only volatile context.
- [ADR-007: Palette views are a Go-owned typed descriptor vocabulary with JSON Patch live updates](adrs/adr-007.md) — no json-render; Adaptive-Cards-style versioning; host renders everything.
- [ADR-008: Programmable palette views run in the extension subprocess over `view.provider`](adrs/adr-008.md) — Raycast/Vicinae event loop on the existing provides-RPC channel; per-client sessions; three-band budget; protocol additions.
- [ADR-009: Two composable view tiers; the programmable tier is TypeScript-only this cycle](adrs/adr-009.md) — declarative tier = projection into the same frame vocabulary; Go MVU deferred; React-in-Go alternatives rejected with evidence.
