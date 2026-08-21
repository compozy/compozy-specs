# UI/UX Change Map: Command Palette — OS-Grade Overhaul

Every UI surface this feature touches: where it lives today, what changes, which states must be designed, and the reference artboard each surface needs. Artboards land under `docs/design/opendesign/command-palette/` — a **ten-file inventory, produced by the operator before execution, outside the task graph** — and become the visual contracts the implementation tasks cite. A UI-bearing task whose artboard is missing at execution time blocks; it never improvises the reference.

Companions: `_spec.md` Part I (behavior authority), `_user_stories.md` (states come from ACs/ECs), `_dx.md` (the non-UI half of the surface).

## Design constraints (apply to every artboard)

- **Locked palette grammar (production parity, binding — supersedes the herdr glass inheritance, 2026-08-18 round 3):** the palette is FLAT (`DESIGN.md` §5 + live `os-command-palette.tsx`): opaque `--color-canvas-soft` panel, `--radius-lg`, `--shadow-overlay` on the floating popup, `--shadow-hairline` on nested popups (action panel, dropdowns, tooltips); the only blur is the 3px `--overlay-blur` scrim; glass belongs to OS-shell chrome (menubar/dock), never the palette. Selection is the `--color-elevated` raise + `text-fg-strong` (production `CommandItem` `data-selected`) — no `--row-selected` plate, no top-light-edge rim, no accent bar; rows carry 18px tinted glyph roundels (`SessionBadgeGlyph` grammar); never state by color alone (glyph + label always).
- **Signal mapping (proposed; design pass finalizes):** destructive actions/confirmations `#E0635A` danger · attention/needs-you follows the existing badge→tone dictionary (no second tone map — anti-duplication rule) · extension-source chips `#8E8EB5` info · success feedback `#5FBF85`.
- **Truthful UI:** availability reasons come from the runtime verbatim; unknown availability renders as disabled-generic ("unavailable right now"), never invented specifics; global-hotkey controls are absent (not disabled) when no desktop shell exists — except in Settings, where the disabled state carries the "requires desktop shell" reason (US-024.AC-4); no fake progress — async feedback reflects real invocation state (US-017).
- **Keyboard/a11y notes:** ARIA combobox pattern throughout (input keeps focus, `aria-activedescendant` selection); dialog focus trap + restore to trigger; ⌘K toggles the action panel open/closed; Esc ladder: panel → confirmation/args → view stack → close; ⌫ pops only on empty query; grid adds ←→ without breaking the ladder; every artboard state reachable keyboard-only.
- **Copy:** `COPY.md` register; canonical names "Command palette" / "palette views"; no subtitles or helper prose under headings; availability reasons are sentence fragments without trailing periods ("needs two windows on this desktop").

## Surface map

| #   | Surface                                | Kind   | Core change                                                        | Stories                     |
| --- | -------------------------------------- | ------ | ------------------------------------------------------------------ | --------------------------- |
| S1  | Palette root                           | modify | Registry-driven groups + pins/recents/context, entities, fallback  | US-001..008, 018..021, 026  |
| S2  | View shell & stack                     | modify | Generalize shell to four view kinds; stack semantics unchanged     | US-009                      |
| S3  | Domain list views (all domains)        | modify | Sessions grammar generalized to every list-bearing domain          | US-010                      |
| S4  | Detail accessory pane                  | new    | Read-only preview split beside list rows                           | US-011                      |
| S5  | Form view                              | new    | Typed structured input inside the palette                          | US-012                      |
| S6  | Grid view                              | new    | Sectioned media grid with 2D keyboard nav                          | US-013                      |
| S7  | Action panel                           | new    | ⌘K-inside-⌘K on the selected row                                   | US-014                      |
| S8  | Inline arguments mode                  | new    | Typed argument fields in the input bar                             | US-015                      |
| S9  | Confirmation step                      | new    | Declared destructive confirmation inside the palette               | US-016                      |
| S10 | Destination mode                       | modify | Same shipped flow, re-sourced from the registry                    | US-036                      |
| S11 | Menubar menus                          | modify | Items become registry projections (label/chord/availability)       | US-035                      |
| S12 | Settings › Shortcuts table             | modify | Whole registry, source filter, aliases, global hotkeys, conflicts  | US-022..024, 029            |
| S13 | Shortcuts cheatsheet                   | modify | Extension-contributed bindings grouped by source                   | US-025                      |
| S14 | Execution feedback (toasts/progress)   | modify | Honest async lifecycle for command invocations                     | US-017                      |
| S15 | Settings › Palette                     | new    | Fallback targets order/enable + personalization reset              | US-021, US-026              |
| S16 | Extensions settings detail             | modify | Palette contributions panel per extension (commands/views/chords)  | US-027..029                 |

### S1. Palette root

- **Today**: `web/src/systems/os/components/os-command-palette.tsx` (root/view branch, single mount at `desktop-shell.tsx:315`); groups hand-written across `os-command-palette-{views,results,window-actions,shell-actions}.tsx`; model in `use-os-command-palette.ts` (343 L); no pins/recents, substring search, no entity sections beyond sessions/tabs/worktrees, no hints on most rows.
- **Change**: one registry-driven result assembly — empty-query state (Pinned → Recents → context group → curated groups), typed entity sections per domain arriving async, settings destinations, ghost autocomplete tail, chord badge on every bound row, "Ask agent: '{query}'" fallback row, scope-globe chip widening every domain.
- **States to design**: rest state with pins/recents (US-005.AC-1); true first-run curated state (US-005.AC-2); query with mixed groups + ghost tail (US-002, US-006.AC-1); zero-match with fallback row only (US-026.AC-1); weak-match with results + fallback row (US-026); global-scope rows with workspace labels (US-007.AC-2); daemon-unavailable degradation (US-001.EC-1); disabled-with-reason rows (US-037.AC-1); async section loading (US-003.AC-4); section error state (US-003.EC-3); overflow note at scale (US-001.EC-2).
- **Artboards**: `command-palette-root.html` (anatomy: rest, first-run, query + ghost, async sections, global-scope labels) + `command-palette-root-states.html` (zero/weak-match fallback, daemon-unavailable, disabled-with-reason, section error, overflow).

### S2. View shell & stack

- **Today**: `os-palette-view-shell.tsx` (generic list chrome), `os-palette-view-stack.tsx` (`PALETTE_VIEW_FRAMES`, one entry), `palette-view-stack.ts` (breadcrumb ≤ 3, selection resolution), `os-palette-breadcrumb.tsx`, `os-palette-footer.tsx`.
- **Change**: shell generalizes to render the four kinds (List keeps today's chrome; Detail/Form/Grid get kind-specific bodies under identical stack chrome); breadcrumb/footer unchanged; "view unavailable" frame for dead ids.
- **States to design**: each kind under the same chrome at depth ≥ 2 (US-009.AC-1/AC-2); re-push same view fresh (US-009.AC-3); view-unavailable frame with named extension (US-009.EC-1, US-028.EC-5); loading and timeout/retry frames for extension-sourced views (US-028.EC-2); programmable-view bands (US-039): busy-with-previous-rows (soft budget), degraded with last-good + inline retry (hard ack), circuit-broken until reopen, program-crashed unavailable variant, "view reloaded" note after dev reload.
- **Artboards**: `command-palette-view-shell.html` (stack chrome, four kinds under identical chrome, unavailable/loading/timeout frames) + `command-palette-view-bands.html` (programmable bands: busy-with-previous-rows, degraded + retry, circuit-broken, crashed, "view reloaded").

### S3. Domain list views (all domains)

- **Today**: Sessions only — `use-os-palette-sessions-view.tsx` (150-row cap), `os-palette-session-chips.tsx`, `os-palette-session-row.tsx`, `palette-session-filters.ts`.
- **Change**: the Sessions grammar (chips with truthful counts, single-select, one-keystroke clear, live refresh with selection survival) becomes the template for every list-bearing domain — sessions, worktrees, tasks, loops, jobs, triggers, agents, bridges, knowledge, vault (names only), network channels, marketplace, extensions. Per-domain chip sets and row anatomy defined per domain; shared status-tone dictionary and attention comparator mandatory.
- **States to design**: one exemplar domain beyond Sessions at full fidelity (tasks: chips All/Queued/Running/Needs-approval/Done + row with state badge); empty-with-filter state (US-010.EC-1); overflow "showing N of M" (US-010.AC-4); cold-cache loading (US-010.EC-3); vault name-only rows (US-010.EC-4).
- **Artboard**: `command-palette-domain-list.html` (tasks exemplar + empty-with-filter + cold-cache + vault names-only).

### S4. Detail accessory pane

- **Today**: none.
- **Change**: optional split pane beside list rows — metadata fields + sanitized rich text; selection-driven; focus never leaves the list.
- **States to design**: populated metadata+text; neutral empty ("no preview"); stale-cleared after row deletion (US-011.EC-1); independent scroll at length (US-011.EC-2); sanitized-degraded plain text (US-011.EC-3).
- **Artboard**: `command-palette-domain-list.html` (shared with S3 — detail pane states).

### S5. Form view

- **Today**: none.
- **Change**: typed fields (text, password, checkbox, dropdown, file/directory) in declared order under stack chrome; inline per-field validation; submit executes the command action; Esc/⌫-on-empty pops discarding values.
- **States to design**: pristine; per-field invalid with focus on first error (US-012.AC-2); submit-failed with preserved values (US-012.EC-2); password masking (US-012.EC-3); empty dropdown with declared hint (US-012.EC-4).
- **Artboard**: `command-palette-form-grid.html`.

### S6. Grid view

- **Today**: none.
- **Change**: sectioned grid, ←→↑↓ navigation, same search/chips/stack contract; ⏎ and action panel parity with rows.
- **States to design**: populated sections; media-failed placeholder tiles (US-013.EC-1); empty grid (US-013.EC-2); overflow/virtualized scale (US-013.EC-3).
- **Artboard**: `command-palette-form-grid.html` (shared).

### S7. Action panel

- **Today**: none (single implicit action per row).
- **Change**: ⌘K opens a filterable panel anchored to the selected row: sections, per-action chord badges, primary marked ↩, destructive styling, meta-actions (Pin/Unpin · Set alias… · Set shortcut…) on every command row, domain actions on entity rows.
- **States to design**: open panel with sections + filter; filtered-empty; destructive action emphasis; disabled-row panel (meta-actions + reason only, US-014.EC-2); panel auto-close on vanished row (US-014.EC-1).
- **Artboard**: `command-palette-action-panel.html`.

### S8. Inline arguments mode

- **Today**: none.
- **Change**: selecting an argument-bearing command morphs the input bar into inline typed fields with placeholders; ⇥ between fields; dropdown popover with type-to-filter; ⏎ executes when required fields filled; Esc restores search.
- **States to design**: fields pristine/filled; required-missing block with focused field (US-015.AC-2); dropdown open (US-015.AC-3); invalid-type message (US-015.EC-2); entered-via-hotkey state (palette opens directly in args mode, US-015.EC-3).
- **Artboard**: `command-palette-args-confirmation.html`.

### S9. Confirmation step

- **Today**: none.
- **Change**: declared confirmations render in-palette: title, body naming the target, confirm verb; Cancel focused by default; repeat-guarded.
- **States to design**: standard confirm; destructive emphasis; invalidated-target state (US-016.EC-2).
- **Artboard**: `command-palette-args-confirmation.html` (shared with S8).

### S10. Destination mode

- **Today**: `paletteIntent.kind === "destination"` threads a boolean through all four group components, hiding three (`os-command-palette-results.tsx` et al.).
- **Change**: same shipped flow re-expressed as a registry query (eligible destinations only); visual behavior unchanged; empty-destination state added.
- **States to design**: destination list; zero-eligible empty state (US-036.EC-2).
- **Artboard**: none — shipped surface re-sourced; `command-palette-root-states.html` covers the frame + the zero-eligible empty state.

### S11. Menubar menus

- **Today**: `web/src/systems/os/components/menubar/{go,session,compozy,window,workspace,help}-menu.tsx` — hand-written items; Window menu already consumes `use-os-window-commands.ts`.
- **Change**: every command item becomes a registry projection (label, chord badge, availability + reason via tooltip/disabled state, same dispatch); grouping/order stays hand-curated per menu.
- **States to design**: none new — existing menu item anatomy gains a chord badge and a disabled-reason treatment consistent with S1; no artboard (structure unchanged; item states inherit S1 grammar).
- **Artboard**: none — covered by tokens/grammar reuse; flagged for design-pass review only if the disabled-reason treatment needs new anatomy.

### S12. Settings › Shortcuts table

- **Today**: `web/src/systems/settings/components/layouts/window-manager-shortcut-table.tsx` + `use-window-manager-shortcut-recorder.ts` — closed ~60-action list, record/conflict/reset, `blocked`/`shadowed` classes.
- **Change**: table lists the entire registry (source filter: Core areas / each extension), adds an Alias column (inline edit, validation), a Global hotkeys section (shell-gated), extension dormant-default rows ("default unavailable — conflicts with X", US-029.AC-2), overwrite flow naming the loser (US-022.AC-2), reset-one/all unchanged.
- **States to design**: source-filtered table; alias editing + invalid alias; conflict block with named culprit + explicit overwrite; overwritten-loser flagged row; global section with per-row "unavailable — in use by another application" (US-024.AC-3) and browser-mode disabled-with-reason (US-024.AC-4); accessibility-permission callout with system deep-link (US-024.EC-1).
- **Artboard**: `command-palette-settings.html`.

### S13. Shortcuts cheatsheet

- **Today**: `os-shortcuts-dialog.tsx` — derives rows from the live effective keymap; surface-local read-only section.
- **Change**: adds extension-sourced groups; multi-chord display unchanged; still fully derived (no static rows).
- **States to design**: none new beyond a source-grouped section — inherits existing dialog anatomy; no artboard.
- **Artboard**: none — derived surface; design-pass review only.

### S14. Execution feedback (toasts/progress)

- **Today**: toast system exists app-wide; palette actions close-and-fire with no async lifecycle.
- **Change**: async invocations surface progress (in-palette affordance while open; toast after close), completion and failure toasts naming command + reason, retry affordance on idempotent-safe failures, "already running" rejection, cross-workspace landing notice.
- **States to design**: in-palette pending affordance; success toast; failure toast with retry; already-running rejection (US-017.EC-2); workspace-switch notice (US-017.EC-3).
- **Artboard**: `command-palette-root-states.html` (pending affordance) — toast anatomy reuses the existing system.

### S15. Settings › Palette

- **Today**: none (no palette settings surface).
- **Change**: new settings section: the **agent fallback toggle** (v1 has exactly one target — no ordered list ships; multi-target ordering arrives only when other target kinds exist, per the settled agent-only direction), personalization master switch mirror (`cmd_palette.personalization`), and "Reset palette personalization" per workspace with scope confirmation. Personalization controls land at P4; the fallback toggle lands **with the fallback behavior at P8** (no control before its runtime exists).
- **States to design**: default state; fallback disabled state (no fallback row renders); reset confirmation; post-reset confirmation feedback.
- **Artboard**: `command-palette-settings-palette.html`.

### S16. Extensions settings detail

- **Today**: `web/src/routes/_app/settings/-extensions-settings-page.tsx` — inventory + instance detail; no palette panel.
- **Change**: per-extension "Palette" panel listing its contributed commands (with effective bindings/dormant defaults) and views; disabled/unhealthy extensions show contributions grayed with the health reason.
- **States to design**: populated panel; dormant-default row; unhealthy-extension gray state (US-027.EC-4).
- **Artboard**: `command-palette-settings-palette.html` (shared with S15).

## Component plan (design → production mapping)

### Rules

- Compose from `@compozy/ui` `Command*` primitives + `Dialog`, `Pill`/`KindChip`/`MonoId`, `StatusDot`/`PillDot`, `Kbd` (`@compozy/ui` exports no generic `Badge` — `Pill` is the chip vocabulary); artboard CSS is a visual contract, never a stylesheet to import. The authoritative class → primitive table is the "Production component map" in `docs/design/opendesign/command-palette/DESIGN-NOTES.md`.
- One registry-consumption path: every surface (root, views, panel, menubar, cheatsheet, settings) renders from the same client-side registry projection — no surface-local command arrays.
- Shared dictionaries are mandatory: status-tone map, attention comparator, chord formatter/glyphs; no view-local forks (anti-duplication rules from herdr bind here).
- Domain views reuse the Sessions row/chips/note primitives (`os-palette-session-*` generalized), not forks per domain.

### New `@compozy/ui` primitives

None expected. `Command*`, `Dialog`, `Badge`, dots, and existing form controls cover every anatomy; the grid body and inline-args bar are domain composites (below), not generic primitives. If the design pass proves a generic gap (e.g. a reusable keycap/chord badge is currently OS-domain-local), the primitive lands in `packages/ui` with story + test per the reuse-before-create rule.

### New domain components (`web/src/systems/os/`)

- `PaletteResults` (replaces the four hand-written group components) — registry projection → sections/rows; used by S1/S10.
- `PaletteEntitySection` — async domain section with loading/error/overflow states; used by S1.
- `PaletteFallbackRow` — the "Ask agent" row + its panel targets; S1.
- `PaletteGhostText` — inline completion tail on the input; S1.
- `PaletteViewFrame` variants: `PaletteListView` (generalized from sessions view), `PaletteDetailPane`, `PaletteFormView`, `PaletteGridView`, `PaletteViewUnavailable`; S2–S6.
- `PaletteActionPanel` (+ `PaletteActionPanelItem`); S7.
- `PaletteArgsBar` (inline argument fields + dropdown popover); S8.
- `PaletteConfirmation`; S9.
- `MenubarCommandItem` — registry-projected menu item (label/chord/availability); S11.
- Settings: `ShortcutSourceFilter`, `AliasCell`, `GlobalHotkeySection`, `PaletteSettingsSection`, `ExtensionPalettePanel`; S12/S15/S16.

### Signal & state mapping

| Design state/glyph                     | Primitive + token                                              |
| -------------------------------------- | -------------------------------------------------------------- |
| Selected row                           | `bg-elevated` + `text-fg-strong` raise (production `CommandItem` `data-selected`; no accent bar, no rim) |
| Row glyph roundel                      | 18px tinted roundel (`SessionBadgeGlyph`), tone from shared dictionary; running pulses on `--color-accent-tint` |
| Destructive action / confirm           | danger `#E0635A` text + glyph; confirm button danger emphasis   |
| Needs-you / attention badge            | existing badge→tone dictionary (danger), unchanged              |
| Extension source chip                  | info `#8E8EB5` chip with extension name                         |
| Disabled-with-reason                   | reduced-contrast row + structured reason hint (no label prose)  |
| Success feedback (toast)               | success `#5FBF85` glyph + label                                 |
| Pending/async affordance               | motion token per `DESIGN.md`; never fake progress percentages   |
| Chord badge / keycap                   | existing `CommandShortcut` + shared glyph formatter             |
