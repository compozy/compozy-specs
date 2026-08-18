# User Stories: Command Palette — OS-Grade Overhaul

Canonical behavior catalog for the command-palette overhaul. Companion to `_spec.md`; consumed by
`_spec.md` Part II (component mapping), `_uiux.md` (surface states), and `_tests.md` (coverage matrix).

Lineage: this spec extends the shipped herdr-parity palette behavior (archived stories herdr US-029/030/031 — view stack, Sessions view, Show-all scope). Shipped semantics are restated here only where this spec changes or generalizes them; unchanged inherited behavior stays authoritative in the archive and in the live QA scenarios.

## Personas

- **Operator** — keyboard-first human running the CompozyOS shell daily; expects Raycast-grade speed, coverage, and personalization from ⌘K.
- **Extension author** — builds CompozyOS extensions; wants commands, views, and default shortcuts in the palette declaratively, without writing renderer code.
- **Agent** — an AI session (or script) managing CompozyOS through CLI/HTTP/UDS and native tools; needs to enumerate and invoke the same commands the operator sees.

## Story Index

| ID     | Feature Area        | Persona          | Story                                                        |
| ------ | ------------------- | ---------------- | ------------------------------------------------------------ |
| US-001 | Catalog & Search    | Operator         | Full OS command catalog in one palette                       |
| US-002 | Catalog & Search    | Operator         | Fuzzy search with stable, explainable ranking                |
| US-003 | Catalog & Search    | Operator         | Entity search across every daemon domain                     |
| US-004 | Catalog & Search    | Operator         | Settings pages and keys are searchable                       |
| US-005 | Catalog & Search    | Operator         | Empty-query surface: pins, recents, context                  |
| US-006 | Catalog & Search    | Operator         | Ghost autocomplete                                           |
| US-007 | Catalog & Search    | Operator         | One workspace-scope model (globe) for all results            |
| US-008 | Context             | Operator         | Context-aware commands from daemon truth                     |
| US-009 | Views               | Operator         | Generalized view stack navigation                            |
| US-010 | Views               | Operator         | A curated view for every list-bearing domain                 |
| US-011 | Views               | Operator         | Detail accessory (preview pane)                              |
| US-012 | Views               | Operator         | Form views for structured input                              |
| US-013 | Views               | Operator         | Grid views for media-shaped results                          |
| US-014 | Action Panel        | Operator         | Per-row action panel (⌘K inside the palette)                 |
| US-015 | Execution           | Operator         | Inline typed arguments                                       |
| US-016 | Execution           | Operator         | Declared confirmation on destructive commands                |
| US-017 | Execution           | Operator         | Honest execution feedback                                    |
| US-018 | Personalization     | Operator         | Frecency ranking that evolves with use                       |
| US-019 | Personalization     | Operator         | Adaptive query learning                                      |
| US-020 | Personalization     | Operator         | Pinned commands                                              |
| US-021 | Personalization     | Operator         | Recents group and personalization reset                      |
| US-022 | Keyboard            | Operator         | Rebind any command                                           |
| US-023 | Keyboard            | Operator         | Aliases (typed keywords)                                     |
| US-024 | Keyboard            | Operator         | OS-global hotkeys through the Electron shell                 |
| US-025 | Keyboard            | Operator         | Shortcut hints and live cheatsheet                           |
| US-026 | NL Fallback         | Operator         | "Ask the agent" fallback with configurable targets           |
| US-027 | Extensions          | Extension author | Declare palette commands in the manifest                     |
| US-028 | Extensions          | Extension author | Declare palette views (two tiers, List/Detail/Form/Grid)     |
| US-029 | Extensions          | Extension author | Declare default shortcuts                                    |
| US-030 | Extensions          | Extension author | Extension commands inherit tool policy                       |
| US-031 | Extensions          | Extension author | Dev-mode hot reload of palette contributions                 |
| US-032 | Agent Control Plane | Agent            | Enumerate the registry with structured output                |
| US-033 | Agent Control Plane | Agent            | Invoke commands by id under policy                           |
| US-034 | Agent Control Plane | Agent            | Manage bindings, aliases, and personalization state          |
| US-035 | Surface Consistency | Operator         | Menubar menus derive from the registry                       |
| US-036 | Surface Consistency | Operator         | Destination mode survives, registry-driven                   |
| US-037 | Surface Consistency | Operator         | Availability honesty (disabled rows say why)                 |
| US-038 | Extensions          | Extension author | Write a programmable view in TypeScript (React DX)           |
| US-039 | Extensions          | Operator         | Live extension views stay instant, honest, and isolated      |

## Catalog & Search

### US-001: Full OS command catalog in one palette

**As an** Operator, **I want** every CompozyOS action reachable from ⌘K, **so that** the palette is the OS's primary control surface instead of a partial mirror.

Acceptance criteria:

- AC-1: Given the palette is open, when I browse or search, then every registered window-manager action (window lifecycle, tiling, tabs, desktops, focus/nav, layout, sessions, workspaces, sidebar, cheatsheet) appears as a command row — no action is keyboard-only.
- AC-2: Given the palette is open, when I search for any OS app, then an "Open {app}" command exists for every app in the catalog.
- AC-3: Given any palette command row, when I open its action panel, then "Set shortcut…" is offered — every command has a stable, bindable id (no palette-only unbindable rows).
- AC-4: Given a command exists in the palette, when it is rendered anywhere else (menubar, cheatsheet, settings shortcut table), then id, label, and effective chord are identical.

Edge cases:

- EC-1: Daemon connection lost while the palette is open → action commands render disabled with a "runtime unavailable" reason; availability-exempt commands (e.g. open cheatsheet) keep working; recovery re-enables rows without reopening.
- EC-2: Catalog at full scale (hundreds of command rows) → sections stay ordered, keyboard navigation stays responsive, and no group renders unbounded without scroll or an exact overflow note.
- EC-3: A command's backing feature is disabled by config → the row is hidden entirely (fully irrelevant), not rendered disabled.
- EC-4: Two registrations claim the same command id → the daemon rejects the later registration at load with a diagnostic; the palette never shows duplicate ids.

### US-002: Fuzzy search with stable, explainable ranking

**As an** Operator, **I want** typo-tolerant search with predictable ordering, **so that** the right command is under ⏎ on the first keystrokes.

Acceptance criteria:

- AC-1: Given the query partially matches title, alias, or keywords, when results render, then matches are case-insensitive, diacritic-insensitive, and tolerate word-boundary/subsequence typing ("nwt" finds "New tab").
- AC-2: Given multiple matches, when results render, then ordering is deterministic for identical input and personalization state (stable across re-renders).
- AC-3: Given results from different groups (commands, entities, settings), when results render, then group precedence is fixed — entity results never interleave inside the command group and vice versa.
- AC-4: Given an alias match, when the row renders, then it shows "Title (alias)" so the vocabulary is taught back.

Edge cases:

- EC-1: Query contains regex/special characters (`(`, `*`, `\`) → treated as literals; no crash, no filter corruption.
- EC-2: Query longer than any candidate or > 256 chars → empty result set with the NL fallback row (US-026); input never truncates silently.
- EC-3: Results arrive in async waves (entities slower than commands) → later waves merge without reordering already-rendered higher-precedence groups and without stealing keyboard selection.
- EC-4: Every query term matching nothing → candidate dropped entirely (no weak "contains one term" noise).

### US-003: Entity search across every daemon domain

**As an** Operator, **I want** to find any live entity — sessions, open tabs, worktrees, agents, tasks, loops, jobs, triggers, bridges, knowledge docs, vault entries, network channels, marketplace items — by typing its name, **so that** navigation never requires knowing which app owns it.

Acceptance criteria:

- AC-1: Given entities exist in the current workspace, when I type ≥ 2 matching characters, then matching entities appear in typed sections with a domain glyph and state badge where the domain has states.
- AC-2: Given I select a session result, when I press ⏎, then I land on that session exactly like the Sessions view does today (window focus, restore if closed, workspace switch first for foreign sessions, `done` marked seen on arrival) — one landing path, not two.
- AC-3: Given I select a non-session entity, when I press ⏎, then I land on its owning app route with the entity focused/selected.
- AC-4: Given a domain fetch has not resolved, when results render, then its section shows a loading state without blocking other sections.

Edge cases:

- EC-1: Entity deleted between listing and ⏎ → honest error ("no longer exists"), row removed on next refresh; no dead navigation.
- EC-2: A domain at 100× typical volume → its section caps visible rows with an exact "showing N of M" note; other sections unaffected.
- EC-3: A domain endpoint errors → that section renders an inline error state naming the domain; other sections unaffected.
- EC-4: Vault entries → titles/names only; secret values never appear in search results, previews, or match highlights.
- EC-5: Two entities with identical names in different workspaces (global scope on) → both render with per-row workspace labels.

### US-004: Settings pages and keys are searchable

**As an** Operator, **I want** to type a setting's name and jump to it, **so that** the ~25 settings routes stop being a memory exercise.

Acceptance criteria:

- AC-1: Given the palette is open, when I search a settings page name, then a "Settings → {page}" result navigates to that page.
- AC-2: Given I search a known configuration concept (e.g. "shortcuts", "theme", "sandbox"), when results render, then keyword-matched settings destinations appear even when the page title differs.

Edge cases:

- EC-1: A settings page requires a state I don't have (e.g. no shell for shell-only pages) → the result is hidden, not broken navigation.
- EC-2: Settings search vs command name collision ("shortcuts" matches both the cheatsheet command and the settings page) → both appear in their own groups; group precedence disambiguates.

### US-005: Empty-query surface: pins, recents, context

**As an** Operator, **I want** the open-palette state to already show what I likely want, **so that** frequent flows are ⌘K + ⏎.

Acceptance criteria:

- AC-1: Given a fresh open with an empty query, when the root renders, then order is: pinned commands, recents, then curated groups (views, apps, context-relevant commands).
- AC-2: Given no pins and no history (first run), when the root renders, then curated defaults render — the palette is never empty at rest.

Edge cases:

- EC-1: A pinned/recent command no longer exists (extension removed, action retired) → the row is silently dropped from the group; stored state is tolerantly pruned, never an error.
- EC-2: Recents contain commands now unavailable in context → rendered disabled with reason (US-037), not hidden, so muscle memory isn't gaslit.

### US-006: Ghost autocomplete

**As an** Operator, **I want** an inline completion hint, **so that** long command names cost three keystrokes.

Acceptance criteria:

- AC-1: Given the top result is high-confidence, when I type, then the rest of its title renders as ghost text after the caret; pressing → at end-of-input accepts it into the query.
- AC-2: Given confidence is low or results are ambiguous, when I type, then no ghost text renders.

Edge cases:

- EC-1: Ghost suggestion case differs from typed prefix → typed characters keep operator casing; only the completion tail renders.
- EC-2: → pressed with the caret mid-input → normal caret movement; acceptance only triggers at end-of-input.

### US-007: One workspace-scope model (globe) for all results

**As an** Operator, **I want** the same current-workspace/all-workspaces toggle everywhere in the palette, **so that** scope is one learned concept.

Acceptance criteria:

- AC-1: Given default state, when I search entities, then results are scoped to the current workspace.
- AC-2: Given I widen scope (globe chip), when results render, then all entity domains and views widen together, rows carry workspace labels, and the preference persists through the same daemon-backed preference the sidebar and Sessions view already share.
- AC-3: Given scope is widened, when I reopen the palette later (any tab/device), then the widened scope is still in effect.

Edge cases:

- EC-1: A workspace is deleted while its rows are listed under global scope → rows vanish on refresh; selecting a stale row yields the honest US-003.EC-1 error.
- EC-2: Single-workspace installation → globe still present and functional; widened scope equals current scope without error.

## Context

### US-008: Context-aware commands from daemon truth

**As an** Operator, **I want** the palette to reflect what is actually focused and running, **so that** relevant commands surface first and irrelevant ones don't exist.

Acceptance criteria:

- AC-1: Given a focused window, when the palette opens, then commands addressing the focused window (close, zoom, tile, move-to-desktop) are available and boosted; with no focused window they are disabled with reason or hidden per US-037.
- AC-2: Given a focused session window, when the palette opens, then that session's contextual commands (interrupt, jump, session ops) rank above global commands for matching queries.
- AC-3: Given context predicates, when they evaluate, then they evaluate against daemon/runtime truth (session state, workspace, window topology, shell presence), never against SPA route strings alone.

Edge cases:

- EC-1: Context changes while the palette is open (session finishes, window closes) → availability and boosts re-evaluate live without stealing keyboard selection (nearest-neighbor rule).
- EC-2: Contradictory context (focused window id points at a just-closed window) → commands degrade to disabled-with-reason; no crash, no ghost targeting.
- EC-3: Context evaluation unavailable (daemon reconnecting) → context commands disable with the US-001.EC-1 reason; context never silently defaults to "everything allowed".

## Views

### US-009: Generalized view stack navigation

**As an** Operator, **I want** the shipped nested-view mechanics to hold for every view kind, **so that** navigation is one contract at any depth.

Extends herdr US-029 (shipped): push/pop stack, per-level scoped search, breadcrumb ≤ 3 slots left-truncating, ⌫ pops only on empty query, Esc closes the whole stack, reopen starts at root, no result bleed across levels, live refresh never steals selection.

Acceptance criteria:

- AC-1: Given any view kind (List, Detail, Form, Grid), when pushed, then the same stack semantics above apply unchanged.
- AC-2: Given a view pushes another view (list row → detail, list row → form), when I pop, then I return to the exact parent state (query, chips, selection preserved).
- AC-3: Given the same view id is pushed again at a deeper level, when it mounts, then it mounts fresh (no state inherited from the earlier instance).

Edge cases:

- EC-1: View id in the stack becomes unknown (extension disabled mid-navigation) → that frame renders an honest "view unavailable" state; popping continues to work; reopen starts at root.
- EC-2: Stack at unusual depth (5+) → breadcrumb keeps the ≤ 3-slot left-truncating contract; performance stays responsive.
- EC-3: Destination intent and view stack remain mutually exclusive (inherited) → opening destination mode with a stack present resets the stack, and vice versa.

### US-010: A curated view for every list-bearing domain

**As an** Operator, **I want** a registered palette view per daemon list domain (sessions, worktrees, tasks, loops, jobs, triggers, agents, bridges, knowledge, vault, network channels, marketplace, extensions), **so that** browsing and filtering any domain is a palette-native experience.

Acceptance criteria:

- AC-1: Given the root Views group or a domain command, when I enter a domain view, then it offers domain-appropriate filter chips with truthful counts, single-select semantics, and one-keystroke clear on zero matches (Sessions-view grammar generalized).
- AC-2: Given a domain with states (tasks, loops, jobs, sessions), when rows render, then state chips/badges use the shared status-tone dictionary — no view-local tone maps (anti-duplication rule).
- AC-3: Given a domain with an attention concept, when rows order, then attention-first ordering uses the shared comparator.
- AC-4: Given any domain view, when rows exceed the mount cap, then the view reports the exact overflow ("showing N of M") or scrolls virtually — never silent truncation.

Edge cases:

- EC-1: Domain empty → honest empty state naming the active filter ("No failed loops"), with one-keystroke filter clear.
- EC-2: Domain list changes underneath an open view → rows refresh live; selection survives per the shipped resolution rule.
- EC-3: A domain view opened with the domain's app never visited (cold cache) → loading state, then rows; no blank flash treated as empty.
- EC-4: Vault domain view → names/metadata only; values never render (matches US-003.EC-4).

### US-011: Detail accessory (preview pane)

**As an** Operator, **I want** a read-only preview beside list rows, **so that** I can inspect before I land.

Acceptance criteria:

- AC-1: Given a list view row with detail content, when the row is selected, then a detail pane renders beside the list (metadata fields and/or sanitized rich text) without moving keyboard focus out of the list.
- AC-2: Given a row without detail content, when selected, then the pane is absent or shows a neutral empty state — never stale content from the previous row.

Edge cases:

- EC-1: Detail content references an entity deleted mid-view → pane clears to empty state on refresh.
- EC-2: Detail content exceeds the pane → pane scrolls independently; list keyboard contract unchanged.
- EC-3: Hostile markdown/content from an extension detail payload → sanitized (no script, no layout escape); rendering failure degrades to plain text, not a broken pane.

### US-012: Form views for structured input

**As an** Operator, **I want** typed form input inside the palette, **so that** structured command input doesn't force a context switch to an app page.

Acceptance criteria:

- AC-1: Given a command whose input is form-declared, when invoked, then a form view pushes with typed fields (text, password, checkbox, dropdown, file/directory where applicable), keyboard-traversable in declared order.
- AC-2: Given required fields are empty or invalid, when I submit, then submission blocks with inline per-field messages; focus moves to the first invalid field.
- AC-3: Given a valid submit, when the command executes, then execution follows the command's action and US-017 feedback rules.

Edge cases:

- EC-1: Esc/⌫-on-empty pops the form → entered values are discarded; a re-push starts clean (no half-submitted state).
- EC-2: Submit fails downstream (policy denial, runtime error) → the form stays open with the error surfaced; entered values preserved.
- EC-3: Password-typed fields → masked rendering, excluded from any logging/echo, cleared on pop.
- EC-4: Dropdown with zero options (dynamic source empty) → field renders its declared empty hint; required+empty blocks submit with an honest reason.

### US-013: Grid views for media-shaped results

**As an** Operator, **I want** grid rendering for icon/media catalogs (marketplace, appearance assets), **so that** visual choices are made visually.

Acceptance criteria:

- AC-1: Given a grid view, when it renders, then items render in a sectioned grid with ↑↓←→ two-dimensional keyboard navigation and the same search/filter/stack contract as lists.
- AC-2: Given a grid item is selected, when I press ⏎ or open the action panel, then behavior matches list rows (primary action, panel actions).

Edge cases:

- EC-1: Item media fails to load → glyph/placeholder fallback with the title visible; selection and actions unaffected.
- EC-2: Empty grid → same honest empty-state grammar as lists.
- EC-3: Grid at large scale → capped or virtualized per US-010.AC-4; arrow navigation stays O(1) responsive.

## Action Panel

### US-014: Per-row action panel (⌘K inside the palette)

**As an** Operator, **I want** secondary actions on the selected row, **so that** one search reaches every operation on a target, not just the default one.

Acceptance criteria:

- AC-1: Given any selected row (command or entity), when I press ⌘K inside the palette, then an action panel opens listing that row's actions in sections, filterable by typing, with per-action shortcut badges; ⌘K again or Esc closes it.
- AC-2: Given the panel is closed, when I press ⏎, then the primary action runs — the primary action is visibly marked on the panel (↩).
- AC-3: Given any command row, when the panel opens, then meta-actions are always present: Pin/Unpin, Set alias…, Set shortcut….
- AC-4: Given an entity row, when the panel opens, then domain actions for that entity (e.g. session: land, interrupt, new tab here; worktree: scope to it; task: open, run) are listed with destructive ones styled as destructive.

Edge cases:

- EC-1: Row vanishes while its panel is open (live refresh) → panel closes; selection falls to nearest neighbor; no action fires against a dead target.
- EC-2: Panel opened on a disabled row → panel opens showing only meta-actions plus the disabled reason; unavailable actions are not listed as runnable.
- EC-3: Action shortcut pressed while panel focus is elsewhere in the palette → the shortcut still fires for the selected row (capture-level dispatch), guarded against key repeat.

## Execution

### US-015: Inline typed arguments

**As an** Operator, **I want** commands to take arguments in the input bar, **so that** the common case skips a form.

Acceptance criteria:

- AC-1: Given a command declaring inline arguments (text, password, dropdown), when I select it, then argument fields render inline in the input bar with placeholders; ⇥ moves between fields; ⏎ executes when required fields are filled.
- AC-2: Given required arguments are missing, when I press ⏎, then execution blocks and the first empty required field focuses with its placeholder emphasized.
- AC-3: Given a dropdown argument, when focused, then its options render for selection with type-to-filter.

Edge cases:

- EC-1: Esc during argument entry → returns the command to normal search state; entered argument values discarded.
- EC-2: Pasted value violating the argument type (text into number-typed) → field-level validation message; execution blocked until fixed.
- EC-3: Command with arguments invoked via its bound hotkey → the palette opens directly in argument mode for that command.
- EC-4: Password-typed argument → masked, never echoed to history/personalization (query learning stores the pre-selection query only, never argument values).

### US-016: Declared confirmation on destructive commands

**As an** Operator, **I want** destructive commands to declare a confirmation step, **so that** irreversibility is a contract, not a hope.

Acceptance criteria:

- AC-1: Given a command declared destructive, when executed from any surface (palette, hotkey, menubar, action panel), then a confirmation renders inside the palette naming the target and the exact effect, with Cancel focused by default.
- AC-2: Given the confirmation is open, when I press Esc, then nothing executes and I return to the prior state.
- AC-3: Given I confirm, when execution proceeds, then US-017 feedback rules apply.

Edge cases:

- EC-1: ⏎ held or double-pressed from the triggering keystroke → the confirmation absorbs it (repeat-guarded); confirming requires a distinct activation on the confirm control.
- EC-2: Target state changed between trigger and confirm (e.g. window already closed) → confirmation invalidates with an honest message instead of executing against the wrong target.
- EC-3: Confirmation triggered by an agent invocation (US-033) → the agent path receives the structured confirmation requirement (approval flow); the UI confirmation is never silently bypassed.

### US-017: Honest execution feedback

**As an** Operator, **I want** every execution to report what happened, **so that** the palette never fails silently.

Acceptance criteria:

- AC-1: Given a synchronous UI command (navigate, window op), when it succeeds, then the palette closes and the effect is visible immediately.
- AC-2: Given an asynchronous command (tool invocation, extension command), when it starts, then a progress affordance is visible (in-palette while open, toast after close); completion and failure both report, with failures naming the command and the reason.
- AC-3: Given a failed execution, when the failure toast renders, then it offers retry where the action is idempotent-safe.

Edge cases:

- EC-1: Daemon connection drops mid-invocation → failure reported with the runtime-unavailable reason; no optimistic success.
- EC-2: Same command invoked again while the first invocation is in flight → non-idempotent commands debounce (second invoke rejected with "already running"); idempotent toggles execute normally.
- EC-3: A command's effect lands in another workspace (cross-workspace landing) → feedback names the workspace switch; no silent context jump.

## Personalization

### US-018: Frecency ranking that evolves with use

**As an** Operator, **I want** frequently and recently used commands to rank higher, **so that** my habits shorten my paths.

Acceptance criteria:

- AC-1: Given I execute a command repeatedly across days, when I search a query matching it and peers, then it ranks above textual-score peers with no usage.
- AC-2: Given I stop using a command, when weeks pass, then its boost decays toward neutral (half-life behavior).
- AC-3: Given identical query and personalization state, when I search twice, then ordering is identical (determinism preserved under personalization).

Edge cases:

- EC-1: Fresh install / empty history → ranking falls back to curated tier order; no cold-start emptiness.
- EC-2: Personalization store corrupt or unreadable → palette works with defaults and reports the degraded state once; store is repaired/reset tolerantly, never crashes search.
- EC-3: Usage recorded for a command that later disappears → stale entries are pruned tolerantly on read (legacy tolerant-read lesson).

### US-019: Adaptive query learning

**As an** Operator, **I want** the palette to learn what my queries mean, **so that** "gh" resolves to what I always pick for "gh".

Acceptance criteria:

- AC-1: Given I search a query and launch a result, when I later type the same or a prefix-related query, then that result receives a strong learned boost.
- AC-2: Given learned associations exist, when they age past their half-life without reinforcement, then their influence decays.
- AC-3: Given personalization is workspace-scoped, when I switch workspace, then learned associations from other workspaces do not leak into ranking.

Edge cases:

- EC-1: Learned target no longer exists → association pruned on read; no ghost boost.
- EC-2: Two different targets learned for the same query at different times → the more recent/reinforced association dominates; ordering remains deterministic.
- EC-3: Password/argument values (US-015.EC-4) → never stored; only the pre-selection query string is learned.

### US-020: Pinned commands

**As an** Operator, **I want** to pin commands, **so that** my chosen few are always first.

Acceptance criteria:

- AC-1: Given a command row, when I Pin via the action panel, then it appears in the Pinned group at the top of the empty-query root, ordered by pin time.
- AC-2: Given a pinned command, when I Unpin, then it leaves the group immediately.
- AC-3: Given pins exist, when I search (non-empty query), then pins rank by normal scoring (pinning is placement at rest, not a search-score cheat).

Edge cases:

- EC-1: Pinned target disappears (US-005.EC-1) → pruned silently.
- EC-2: Pinning the same command twice (e.g. from two surfaces quickly) → idempotent; one pin entry.

### US-021: Recents group and personalization reset

**As an** Operator, **I want** a Recents group and a way to reset ranking state, **so that** the palette stays trustworthy and correctable.

Acceptance criteria:

- AC-1: Given commands were executed, when I open the palette with an empty query, then a Recents group lists recent commands (deduplicated, most recent first); "open palette" itself is never recorded.
- AC-2: Given personalization state exists, when I reset it from Settings, then pins/recents/frecency/query-learning clear per chosen scope (this workspace), and the root returns to curated defaults.

Edge cases:

- EC-1: Reset while a palette is open in another tab → the other tab reflects the reset on next open/refresh (daemon-persisted state, no tab-local ghosts).
- EC-2: Recents rendered for commands now context-unavailable → US-005.EC-2 behavior.

## Keyboard

### US-022: Rebind any command

**As an** Operator, **I want** to bind or rebind a chord on any command — core or extension — **so that** my keymap is mine.

Acceptance criteria:

- AC-1: Given the Settings shortcut table, when it renders, then it lists every registry command (filterable by source: core areas, extensions) with its effective binding; palette rows deep-link here via "Set shortcut…".
- AC-2: Given I record a chord on a command, when it conflicts with another binding, then the assignment is blocked with the culprit named ("already used by 'X'"), offering explicit overwrite; on overwrite the loser becomes unbound and is flagged in the table.
- AC-3: Given a rebind succeeds, when I use the chord anywhere in the shell, then it dispatches immediately (no reload), and palette rows/cheatsheet/menubar all show the new chord.
- AC-4: Given overrides exist, when I reset one or reset all, then daemon defaults restore.

Edge cases:

- EC-1: Recording captures a chord shadowed by a surface-local binding (composer Bold class) → the existing shadowed-warning applies, extended to any new surface-local keys this spec adds.
- EC-2: Recording a bare single letter with no modifier → allowed only for commands flagged safe-in-editable-exempt contexts; otherwise rejected with the reason (typing-guard contract).
- EC-3: Stored override references a command id that no longer exists → tolerated on read (dropped with a diagnostic), never a boot failure.
- EC-4: Two tabs edit bindings concurrently → daemon persistence serializes; the later write wins; both tabs converge on the effective keymap without reload.

### US-023: Aliases (typed keywords)

**As an** Operator, **I want** short typed aliases per command, **so that** my vocabulary beats fuzzy matching.

Acceptance criteria:

- AC-1: Given a command, when I set an alias (action panel or settings), then typing the alias ranks that command first, rendered as "Title (alias)".
- AC-2: Given an alias exists, when I try to assign the same alias to another command, then the assignment blocks naming the current owner, with explicit overwrite.
- AC-3: Given an alias is exactly typed, when results render, then the exact-alias match outranks every non-exact result.

Edge cases:

- EC-1: Alias format — 1–32 chars, no whitespace; invalid input rejected inline with the rule stated.
- EC-2: Alias equals another command's full title → alias wins for its owner; the titled command still matches by title; both render (alias owner first).
- EC-3: Alias's command removed → alias mapping pruned tolerantly.

### US-024: OS-global hotkeys through the Electron shell

**As an** Operator, **I want** system-wide hotkeys (summon Compozy, run a command) when running the desktop shell, **so that** the palette behaves like an OS launcher, not a browser tab.

Acceptance criteria:

- AC-1: Given the Electron shell is running, when I press the global summon chord (default assigned; shown and rebindable in Settings), then the Compozy window focuses/restores with the palette open — from any app.
- AC-2: Given a command with a global hotkey assigned, when I press it with Compozy unfocused, then the window summons and the command executes (or opens in argument mode per US-015.EC-3).
- AC-3: Given a global registration fails (chord owned by another app), when Settings renders that binding, then it shows an explicit "unavailable — in use by another application" state; the previous working binding is restored (never left unbound silently).
- AC-4: Given the web app runs without the shell (plain browser), when Settings renders global-hotkey controls, then they are disabled with the "requires desktop shell" reason; in-app chords are unaffected.

Edge cases:

- EC-1: macOS requires Accessibility trust for some chords → detected at shell launch; Settings surfaces a deep-link to System Settings instead of failing silently.
- EC-2: Shell quits/crashes → OS-global registrations are released; on relaunch they re-register and re-report failures per AC-3.
- EC-3: Non-QWERTY layout (known Electron globalShortcut limitation) → binding recorded by physical key; the limitation is surfaced in the recorder UI copy for affected layouts.
- EC-4: Global chord fired while a modal/native dialog is open in Compozy → summon focuses the window without executing through the modal; command execution respects current availability rules.

### US-025: Shortcut hints and live cheatsheet

**As an** Operator, **I want** the palette and cheatsheet to teach the keyboard, **so that** every displayed chord is real.

Acceptance criteria:

- AC-1: Given any row whose command has an effective binding, when it renders (palette, action panel, menubar), then the chord badge shows the live effective binding — including user overrides and extension commands.
- AC-2: Given the cheatsheet (`?`/⌘/), when it renders, then it derives every row from the live keymap, including extension-contributed bindings grouped by source, with surface-local bindings in the read-only reference section.

Edge cases:

- EC-1: Binding changed while palette open → badges update on next render pass; no stale chords.
- EC-2: Command with multiple chords → primary chord badges the row; all chords appear in the cheatsheet/settings.

## NL Fallback

### US-026: "Ask the agent" fallback with configurable targets

**As an** Operator, **I want** unmatched queries to become agent prompts, **so that** a dead end becomes delegation.

Acceptance criteria:

- AC-1: Given zero or weak results for a non-empty query, when results render, then a fallback row "Ask agent: '{query}'" appears; ⏎ sends the query as the opening prompt of a new session with the workspace's default agent, closes the palette, and opens the session window.
- AC-2: Given the palette settings, when I disable the agent fallback, then no fallback row renders; re-enabling restores it. (v1 ships exactly one target — the agent; multi-target ordering and secondary targets on the row's panel arrive only with future target kinds.)
- AC-3: Given nothing is typed, when the root renders, then no fallback row exists.
- AC-4: Given any text was typed, when I read the fallback row, then it is visibly distinct from command results (delegation, not execution), and nothing is sent anywhere before ⏎.

Edge cases:

- EC-1: No default agent resolvable for the workspace → the fallback row opens agent selection instead of failing; selection proceeds to session creation.
- EC-2: Session spawn fails (provider down) → failure toast names the reason; the palette reopens with the query preserved for retry.
- EC-3: Query contains secrets-looking content → sent only on explicit ⏎ (never pre-sent for classification); no local logging of the query beyond standard personalization rules (US-019.EC-3).
- EC-4: Fallback invoked repeatedly with the same query → each ⏎ creates a distinct session (delegation is deliberate, not deduplicated); rapid double-⏎ is repeat-guarded.

## Extensions

### US-027: Declare palette commands in the manifest

**As an** Extension author, **I want** to declare palette commands declaratively, **so that** my extension is present in ⌘K without renderer code.

Acceptance criteria:

- AC-1: Given my manifest declares palette commands (id, title, category/section, icon token, keywords, action), when the extension is enabled in a workspace, then the commands appear in that workspace's palette, namespaced (`ext.<extension>.<command>`), attributed to the extension.
- AC-2: Given the action union, when I declare a command, then I can choose: invoke one of my tools (with arguments mapped from its schema), open one of my contributed views, navigate to an OS app/route, or open an external URL.
- AC-3: Given my extension is disabled or removed, when the palette renders, then my commands are gone; re-enable restores them without daemon restart.
- AC-4: Given an invalid declaration (bad id, unknown tool ref, malformed action), when I run build/validate, then validation fails with an actionable message naming the field — never a silent drop at runtime.

Edge cases:

- EC-1: Command id colliding with a core id or another extension's id → rejected at manifest load with a diagnostic; the palette never shows ambiguous ids.
- EC-2: Declared action references a tool that fails availability (missing binary, unhealthy) → the command renders disabled with the availability reason (US-037), not hidden and not broken.
- EC-3: Hostile strings in titles/keywords (control chars, huge lengths, markup) → sanitized/length-capped at validation; rendering is inert text.
- EC-4: Extension crash-looping → its commands STAY listed, disabled with the health reason ("extension notes is unhealthy (crash loop)"); membership drops only on disable/remove; recovery re-enables without re-registration.
- EC-5: Same extension enabled in two workspaces with different versions (dev overlay) → each workspace's palette reflects its own instance's declarations.

### US-028: Declare palette views (two tiers, List/Detail/Form/Grid)

**As an** Extension author, **I want** to ship palette views declaratively or as programs, **so that** the trivial case costs three manifest lines and the rich case gets full interactivity.

Acceptance criteria:

- AC-1: Given my manifest declares a view (id, title, kind), when an operator opens it (via my command or the Views group), then the host renders it with the exact keyboard/stack contract of built-in views (US-009) and host design tokens — identically for both tiers.
- AC-2: Given a Tier-1 view (`source: {tool}`), when content resolves, then rows/sections/chips/detail/form fields/grid items render from my tool's serialized payload — available to TypeScript and Go extensions alike; my code never executes in the operator's renderer (either tier).
- AC-5: Given a Tier-2 view (`program: true`) in a Go extension, when I run build/validate, then it fails with the actionable message that view programs require a TypeScript extension this release.
- AC-3: Given my view declares row actions, when the operator opens a row's action panel, then my declared actions appear alongside host meta-actions, executing through my declared action union (US-027.AC-2).
- AC-4: Given my payload updates (live data), when the view refreshes, then selection-preservation rules apply exactly as for built-ins.

Edge cases:

- EC-1: Payload fails schema validation → the view renders an honest error frame naming the extension; the palette stays usable; the error reaches extension logs/diagnostics.
- EC-2: Content source slow or unresponsive → loading state, then timeout with retry affordance; the stack never wedges (Esc/⌫ always work).
- EC-3: Oversized payload (rows beyond cap, giant detail body) → capped with exact overflow reporting / truncation markers per US-010.AC-4; never renderer lock-up.
- EC-4: View declared with an unknown kind (future schema version) → rejected at validate for authored manifests; at runtime, an unknown-kind frame degrades honestly ("view requires a newer CompozyOS").
- EC-5: Extension disabled while its view is open → US-009.EC-1 behavior with the extension named.

### US-038: Write a programmable view in TypeScript (React DX)

**As an** Extension author, **I want** to write my view as a React component with hooks, **so that** live search, filters, navigation, and forms are ordinary application code — never wire-format bookkeeping.

Acceptance criteria:

- AC-1: Given `@compozy/extension-react`, when I write JSX with the palette kit (`List`/`List.Item`/`Detail`/`Form`/`Grid`/`ActionPanel`/`Action` + `useNavigation`/`useCachedPromise`/`showToast`) and register it with `registerReactViews`, then the view runs in my extension's subprocess and renders in the palette with full keyboard/stack parity.
- AC-2: Given I pass `onSearchTextChange`, when the operator types, then my handler receives host-throttled text and my re-render reaches the palette as a patch; given I omit it, the host filters my rows locally with zero round trips.
- AC-3: Given `Action.Push` targets and `Form.onSubmit`, when the operator navigates and submits, then pushed components mount fresh with their props, `pop()` returns to the preserved parent state, and submit receives the assembled values.
- AC-4: Given a declared destructive action with `confirmation`, when invoked, then the palette's confirmation step renders before my handler runs (US-016 parity).
- AC-5: Given `compozy extension dev`, when I edit the program, then open sessions drop with the "view reloaded" note and the next open runs the new code (US-031 parity).

Edge cases:

- EC-1: My handler does synchronous CPU work past the yield budget → the SDK surfaces a starvation warning in dev diagnostics; sustained blocking risks my extension's own health probe (documented failure mode), never the palette's.
- EC-2: My program throws during render/event handling → the session ends with the honest unavailable frame naming my extension; reopening starts a fresh session; the error reaches `compozy extension logs`.
- EC-3: My result sets declare `complete: true` → the host filters locally between my updates; stale echoes of controlled values are discarded by event counter — my program can never revert the operator's typing.
- EC-4: The extension restarts (crash loop, update, reload) mid-session → all my view sessions are destroyed; the host reopens fresh on next use; no ghost state survives.

### US-039: Live extension views stay instant, honest, and isolated

**As an** Operator, **I want** extension views to feel native no matter how slow or broken the extension is, **so that** trusting third-party views costs me nothing.

Acceptance criteria:

- AC-1: Given any programmable view, when I type, move selection, scroll, or toggle a chip, then the echo is instant (host-local); round trips only refine results.
- AC-2: Given a slow program, when an update passes the soft budget, then previous rows stay visible with a busy indicator — the list never blanks for a spinner; past the hard ack the view degrades with last-good rows and an inline retry.
- AC-3: Given repeated hard-ack misses, when the third consecutive one occurs, then the view circuit-breaks until I reopen it — Esc/⌫/navigation and every other view keep working throughout.
- AC-4: Given two attached clients (e.g. shell + browser tab) with the same view open, when I search in one, then the other's list does not move — sessions are per client.

Edge cases:

- EC-1: Program crash mid-interaction → unavailable frame naming the extension; palette and other views unaffected; reopening starts fresh.
- EC-2: Daemon-extension connection loss mid-session → same degraded contract as US-001.EC-1: honest reason, recovery re-enables without reopening the palette.
- EC-3: A view session left open when the palette closes → torn down (two-way teardown); no orphan sessions accumulate across opens.

### US-029: Declare default shortcuts

**As an** Extension author, **I want** to suggest default chords for my commands, **so that** my extension is keyboard-first out of the box — without ever stealing the operator's keys.

Acceptance criteria:

- AC-1: Given my command declares a default chord, when the extension enables, then the chord binds only if entirely free (no core default, no user override, no other bound extension default claims it).
- AC-2: Given my default conflicts with anything, when the extension enables, then my command stays unbound, the conflict is visible in the Settings shortcut table ("default unavailable — conflicts with X"), and the operator can bind it manually.
- AC-3: Given an operator has overridden my command's chord, when I ship a new default in an update, then the operator's override wins silently.

Edge cases:

- EC-1: Two extensions enabling with the same free default chord → the first to bind wins deterministically (enable order); the second gets AC-2 treatment.
- EC-2: Extension disabled → its bindings deactivate but user-authored overrides for its commands persist dormant and reactivate on re-enable.
- EC-3: Malformed chord syntax in the manifest → build/validate failure with the chord grammar named.

### US-030: Extension commands inherit tool policy

**As an** Extension author (and Operator), **I want** palette execution to flow through the existing policy machinery, **so that** the palette is a new door to the same guarded room — never a bypass.

Acceptance criteria:

- AC-1: Given a tool-invoking extension command, when executed from the palette, then risk class, approval requirements, availability diagnostics, and workspace trust apply exactly as a direct tool invocation would.
- AC-2: Given approval is required, when the command runs, then the standard approval flow triggers and the palette/toast reflects pending/approved/denied honestly.

Edge cases:

- EC-1: Approval denied → command reports denial as the failure reason; no partial effect.
- EC-2: Workspace not trusted for the extension → command renders disabled with the trust reason before execution is attempted.
- EC-3: Policy changes (tool gated off) while palette open → next render disables the row with reason; in-flight executions resolve under the policy at invocation time.

### US-031: Dev-mode hot reload of palette contributions

**As an** Extension author, **I want** my palette contributions to hot-reload in dev mode, **so that** the palette iteration loop matches the rest of extension dev.

Acceptance criteria:

- AC-1: Given a dev-linked extension, when I change command/view declarations, then the open shell reflects the changes within the dev watch interval without daemon restart or page reload.
- AC-2: Given my dev instance shadows a published instance, when the palette renders in that workspace, then dev declarations win (consistent with the existing dev-overlay rule).

Edge cases:

- EC-1: Reload lands while my view is open → the view refreshes in place when compatible, or pops to root with a "view reloaded" note when not; never a wedged stack.
- EC-2: Reload introduces a validation error → previous good declarations stay live; the error surfaces in dev diagnostics/logs.

## Agent Control Plane

### US-032: Enumerate the registry with structured output

**As an** Agent, **I want** to list the full command registry, **so that** I can discover what the OS can do before doing it.

Acceptance criteria:

- AC-1: Given the CLI/HTTP/UDS surface, when I list commands, then I receive every registry entry (core + extension) with id, title, source, category, availability (with reason), effective bindings, argument schema, and destructive/confirmation flags — in structured output (json/jsonl/toon and human-readable).
- AC-2: Given workspace scoping, when I list with a workspace selector, then extension commands reflect that workspace's enabled instances.
- AC-3: Given a native tool for the same enumeration, when a session calls it, then output parity holds with the CLI/HTTP surface.

Edge cases:

- EC-1: Registry at full scale → listing is complete (no silent caps); pagination/streaming (jsonl) available for large outputs.
- EC-2: Daemon degraded (extension subsystem down) → listing reports partial state explicitly (source-level availability), never a silently shorter list.

### US-033: Invoke commands by id under policy

**As an** Agent, **I want** to invoke palette commands headlessly, **so that** every operator capability is also an agent capability.

Acceptance criteria:

- AC-1: Given a command id and arguments, when I invoke via CLI/HTTP/UDS or the native tool, then execution flows through the same dispatch, policy gates, and confirmation/approval contracts as operator execution, returning a structured result (effect, error, or approval-pending).
- AC-2: Given a UI-effecting command (navigate, window op), when invoked with an attached shell client, then the effect lands in that client; with no attached client, invocation fails fast with an explicit "no attached shell" error — never a fake success.
- AC-3: Given a destructive command, when invoked by an agent, then the declared confirmation maps to the approval flow (US-016.EC-3); no UI-only bypass and no approval-free destruction.

Edge cases:

- EC-1: Unknown id → structured not-found error listing nothing else (no fuzzy guessing on the invoke path).
- EC-2: Arguments failing the declared schema → structured validation error naming fields; nothing executes.
- EC-3: Command disabled in current context → structured unavailable error carrying the same reason string the UI shows.
- EC-4: Concurrent invocations of a non-idempotent command → second receives "already running" (US-017.EC-2 parity).

### US-034: Manage bindings, aliases, and personalization state

**As an** Agent, **I want** to read and adjust palette configuration through the settings surfaces, **so that** setup and hygiene are scriptable.

Acceptance criteria:

- AC-1: Given the settings API/CLI, when I read the keymap, then I get defaults, overrides, extension defaults (bound and dormant), aliases, and conflicts in structured form.
- AC-2: Given a valid change (bind, unbind, alias set/clear), when I apply it, then the same validation/conflict rules as the Settings UI apply, and connected shells reflect it live.
- AC-3: Given personalization state, when I read or reset it (per workspace), then the operation matches US-021.AC-2 semantics.

Edge cases:

- EC-1: Invalid chord/alias grammar via API → structured validation error identical in substance to the UI rule.
- EC-2: Conflicting write (chord already owned) → structured conflict error naming the owner; explicit-overwrite flag required to force.

## Surface Consistency

### US-035: Menubar menus derive from the registry

**As an** Operator, **I want** the menubar to be a projection of the same registry, **so that** no command has three divergent definitions.

Acceptance criteria:

- AC-1: Given the menubar menus (Compozy, Workspace, Session, Window, Go, Help), when they render, then every command item shows the registry's label and effective chord, respects the same availability (disabled with the same reason), and dispatches through the same execution path as the palette.
- AC-2: Given a rebind or an extension enable/disable, when the menubar next renders, then it reflects the change with no code edit or reload.
- AC-3: Given menu curation, when menus render, then item grouping/order remains hand-curated (menus select from the registry; the registry does not dictate menu layout).

Edge cases:

- EC-1: Menu item whose command becomes unavailable → renders disabled with reason; never vanishes mid-session (menus are stable surfaces).
- EC-2: Extension contributing commands mapped into a menu section (if curated in) → disabled extension removes the items at next render without breaking the menu.

### US-036: Destination mode survives, registry-driven

**As an** Operator, **I want** the new-tab destination picker to keep working, **so that** the overhaul does not regress a shipped flow.

Acceptance criteria:

- AC-1: Given a destination intent (new tab / open-in-this-tab), when the palette opens in destination mode, then eligible destination results render (apps, sessions) and ineligible groups are absent, matching shipped behavior on the new registry.
- AC-2: Given destination mode, when I select a destination, then the tab opens/navigates as today and the intent clears.

Edge cases:

- EC-1: Destination intent set while a view stack exists → mutual exclusion holds (stack resets), inherited from shipped behavior.
- EC-2: Destination mode with zero eligible destinations → honest empty state; Esc exits cleanly, intent cleared.

### US-037: Availability honesty (disabled rows say why)

**As an** Operator, **I want** disabled commands to say why, **so that** the palette teaches the system instead of gaslighting me.

Acceptance criteria:

- AC-1: Given a command unavailable in context, when its row renders, then the label stays clean and a structured reason renders as a distinct hint (no "(needs two windows)" prose baked into labels).
- AC-2: Given the fully-irrelevant rule, when context excludes a command entirely (wrong surface, feature off), then it is hidden; partial relevance renders disabled-with-reason.
- AC-3: Given any surface (palette, action panel, menubar, agent error), when the same unavailability occurs, then the same reason text appears.

Edge cases:

- EC-1: Reason unknown (unexpected runtime state) → generic honest fallback ("unavailable right now"), never a fabricated specific reason.
- EC-2: Availability flaps rapidly (reconnect storm) → row state debounces; no flicker between enabled/disabled during a single render burst.
