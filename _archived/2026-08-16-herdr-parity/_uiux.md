# UI/UX Change Map: Herdr Parity (Session Attention · Orchestration DX · Shortcuts v2)

Every UI surface this program touches: where it lives today, what changes, which states were designed, and the locked artboard each surface cites. The design pass is complete — iterate the files in place; never regenerate. Implementation tasks 03/05/06 cite these boards as Visual Contract rows (`eng-ui-screenshot` evidence bundles). Artboard CSS is a visual contract, never a stylesheet to import.

Companions: `_spec.md` Part I (behavior authority), `_user_stories.md` (states come from ACs/ECs), `_dx.md` (the non-UI half of the surface).

## Design set (locked)

- Index: `docs/design/opendesign/herdr-parity/index.html`
- Locked decisions: `docs/design/opendesign/herdr-parity/DESIGN-NOTES.md`
- Domain vocabulary (companion to ds-core + ds-shell; never import): `docs/design/opendesign/herdr-parity/herdr.css`

| Surfaces | Board | Lab sections |
| --- | --- | --- |
| S1 · S2 · S14 | `docs/design/opendesign/herdr-parity/herdr-parity-sidebar.html` | §01 dictionary · §02 Sessions window · §03 sort/scope edges · §04 row edges · §05 status line |
| S3 · S4 | `docs/design/opendesign/herdr-parity/herdr-parity-bell.html` | §01 both sections · §02 section states · §03 badge + tab-title count |
| S5 · S7 | `docs/design/opendesign/herdr-parity/herdr-parity-toasts.html` | §01 stack · §02 variants · §03 fold + resolved race + delivery rules |
| S8 · S6 | `docs/design/opendesign/herdr-parity/herdr-parity-settings-attention.html` | §01 Settings window · §02 system-channel truth + native content table · §03 mute/defaults |
| S9 · S10 | `docs/design/opendesign/herdr-parity/herdr-parity-settings-shortcuts.html` | §01 table · §02 conflicts · §03 Terminal preset |
| S11 | `docs/design/opendesign/herdr-parity/herdr-parity-cheatsheet.html` | §01 sheet · §02 open/freshness/glyph helper |
| S12 · S13 | `docs/design/opendesign/herdr-parity/herdr-parity-palette-sessions.html` | §01 live stack · §02 depth/empty · §03 chips/scope/scale |

## Design constraints (apply to every artboard)

- **No color-only state.** Every badge/state indicator pairs a tone with a distinct glyph/shape and an accessible label (US-001.AC-4). WCAG 1.4.1 is a floor, not a goal.
- **One color, one meaning.** Locked in `DESIGN-NOTES.md`: needs-you class = danger (glyph carries the member); `done` = info + check; `waiting-for-auth` re-inks from warning → danger. Herdr's "teal" in US-002.AC-1 reads as the finished-unseen info tone — no invented tokens.
- **Two scales, one meaning.** 7–9px shapes on rows (circle · diamond · square · check · rings); 18px tinted glyph roundels (`.sig`) on bell rows, palette rows, toasts, and the window status line. The CLI state word is always present.
- **Glass grammar.** Shell-glass (`--shell-glass-pop`) is reserved for floating chrome: bell, toasts, cheatsheet, palette, sort menu. Window content stays on the production radius scale. Selection is a neutral plate (`--row-selected`), never an accent bar.
- **Truthful channels.** The system-notification setting renders its real platform state (unsupported / permission-denied / armed); no pretend-armed toggles (US-012.EC-1/EC-2). Sound failure never surfaces as an error (US-010.EC-2).
- **Calm defaults.** New sections/panels ship closed or quiet: the bell stays a dot+count until opened; Finished never inflates the badge; Show all defaults off.
- **Keyboard-first.** Every new flow (bell rows, palette views, preset preview, cheatsheet) is fully keyboard-operable; jumps manage focus explicitly.
- **Copy.** `COPY.md` register; no helper prose under headings; state names match the CLI vocabulary exactly (`waiting-for-input`, not "asking"). S4 title wording and native-notification copy on the boards are proposals — `COPY.md` owns the final words.
- **Keymap authority.** The frozen default keymap and the terminal preset live in `_dx.md` → Keyboard Defaults; S9 and S11 render that table, never a local copy. Precedence rule: focused UI wins over global chords.

## Surface map

| #   | Surface                              | Kind   | Core change                                                        | Stories                |
| --- | ------------------------------------ | ------ | ------------------------------------------------------------------ | ---------------------- |
| S1  | Sidebar session row                  | modify | New badges (`waiting-for-input`, `done`), unified tone map          | US-001, US-002, US-003 |
| S2  | Sidebar list controls                | modify | Attention-first sort option + Show all (all workspaces) scope       | US-007, US-031         |
| S3  | Attention bell                       | modify | "Needs you" / "Finished" sections, cross-workspace rows, jump       | US-003, US-005, US-014 |
| S4  | Tab/app title count                  | new    | "N awaiting" needs-you count in `document.title`                    | US-006                 |
| S5  | Attention toasts                     | new    | Needs-you toasts, coalesced completion toasts, click-to-jump        | US-008, US-009, US-013 |
| S6  | System notification channel          | new    | OS-level notifications, permission flow, focus gating               | US-012                 |
| S7  | Sound layer                          | new    | Single built-in sound on notification delivery                      | US-010                 |
| S8  | Settings → Attention                 | new    | Toggles (toasts/sound/system) + per-workspace mute                  | US-011, US-015         |
| S9  | Settings → Shortcuts table           | modify | Tabs group added, array/range rendering, conflict diagnostics       | US-024, US-025, US-028 |
| S10 | Terminal preset control              | new    | One-click preset apply with preview diff + revert                   | US-026                 |
| S11 | Shortcuts cheatsheet                 | modify | `?` binding, registry-derived rows, surface-local reference section | US-028                 |
| S12 | Palette nested view mechanism        | modify | View stack, breadcrumb, backspace-pop, view registry                | US-029                 |
| S13 | Palette Sessions view                | new    | State-filter chips, attention-first order, enter-to-land            | US-030                 |
| S14 | Session window header status line    | modify | Same unified tone map as S1 (kills the conflicting duplicate)       | US-001, US-002         |

### S1. Sidebar session row

- **Today**: `web/src/systems/session/components/session-list/session-list-row.tsx:5-25` — `sessionStatusTone()` + `SessionStatusMark` (PillDot, pulse on `running`; StatusDot ring for `stopped`); `data-status` attr at `:54`; badge text as subtitle at `:74`.
- **Change**: render the two new badges; adopt the single exported badge→tone/glyph map (S14 shares it); `done` gets its distinct tone + glyph; needs-you class visually unified.
- **States to design**: `waiting-for-input`, `waiting-for-auth`, `failed` (one class, distinguishable glyphs — US-001.AC-2/AC-4); `done` vs `idle` (US-002.AC-1/AC-3); precedence render when question+permission coexist (US-001.EC-1); `unknown` honesty (US-001.EC-5); `done` yielding to `running` on a new turn (US-002.EC-4).
- **Artboard**: `docs/design/opendesign/herdr-parity/herdr-parity-sidebar.html` — §01 dictionary, §02 window, §04 row edges.

### S2. Sidebar list controls

- **Today**: `web/src/systems/session/components/session-list/session-list.tsx` — `view: "recent" | "all"` toggle (workspace-scoped); sorting server-side via `useSessions(..., sort: "last_activity")`; collapse state persisted in the daemon-backed desktop store (`use-session-sidebar-state.ts`).
- **Change**: add attention-first sort option (persisted per operator); the existing view toggle becomes tri-state — `recent | all | all workspaces` — where the third state widens the list to every workspace, grouped and labeled by workspace.
- **States to design**: attention-first ordering with stable ties (US-007.AC-1/EC-1); transition movement without losing keyboard selection (US-007.EC-2); Show all groups by workspace incl. per-group error and empty states (US-031.AC-1/EC-1); scale at hundreds of sessions (US-031.EC-2); new workspace appearing live (US-031.EC-4).
- **Artboard**: `docs/design/opendesign/herdr-parity/herdr-parity-sidebar.html` — §02 scope pills, §03 sort menu + per-group failure/collapse.

### S3. Attention bell

- **Today**: `web/src/systems/os/components/attention-bell.tsx` (flat "Needs your attention" list: sessions hard-labeled `waiting-for-auth`, tasks `awaiting approval`); count from `attention-model.ts:26` (`badge === "waiting-for-auth"` only); composition in `use-os-attention.ts` (workspace-wide authority, staleness-gated); badge pill `os-menubar.tsx:109-115` (9+ cap).
- **Change**: two sections — **Needs you** (waiting-for-input, waiting-for-auth, failed sessions + existing task approval rows) and **Finished** (`done` sessions); rows carry workspace labels; badge counts needs-you only, cross-workspace; row activation switches workspace when needed and focuses the session window (via the existing open/activate coordinator); arrival at a `done` row marks it seen.
- **States to design**: both sections populated; needs-you-only; finished-only; quiet/empty (US-005.EC-1); disconnected/stale (existing `os-bell-disconnected` state preserved); stale-row click fallback (US-005.EC-2); 100+ rows ordering + scroll (US-005.EC-3); workspace labels/grouping at many workspaces (US-014.EC-3); muted workspace rows still present (US-015.AC-1).
- **Artboard**: `docs/design/opendesign/herdr-parity/herdr-parity-bell.html` — §01 both sections, §02 quiet/disconnected/muted/scale.

### S4. Tab/app title count

- **Today**: none — zero `document.title` management in `web/`.
- **Change**: title carries the cross-workspace needs-you count while > 0 (exact copy from `COPY.md` pass); survives route changes; updates while backgrounded.
- **States to design**: count present / zero (clean title) — text-only; exact number in the title even when the pill caps at 9+.
- **Artboard**: `docs/design/opendesign/herdr-parity/herdr-parity-bell.html` — §03 copy spec. `COPY.md` owns final words.

### S5. Attention toasts

- **Today**: sonner `Toaster` mounted once (`web/src/main.tsx:52`), used only for action feedback via `web/src/lib/user-feedback.ts`. No incoming-state toasts.
- **Change**: needs-you toasts (per-session, immediate, dedup 5s), coalesced completion toasts ("N sessions finished" → opens bell Finished section), agent-sent notification toasts (sender identified); click focuses the session (workspace switch included); suppression per focused session.
- **States to design**: needs-input / needs-auth / failed variants; single completion; grouped completion; agent-sent notification; stacking of 5 simultaneous needs-you toasts (US-008.EC-1); resolved-before-click landing (US-008.EC-4).
- **Artboard**: `docs/design/opendesign/herdr-parity/herdr-parity-toasts.html` — §01 stack, §02 variants, §03 fold + resolved-before-click.

### S6. System notification channel

- **Today**: none — no Notification API usage, no desktop-app notification plugin, no capability entries.
- **Change**: opt-in OS notifications firing only while the app is unfocused/hidden; enabling walks the platform permission flow; activation focuses app → workspace → session.
- **States to design** (in S8's settings context + native chrome): armed / off / permission-denied-with-reason / platform-unsupported (truthful states, US-012.EC-1/EC-2). Native notification rendering is platform chrome — content spec only (title/body/click target).
- **Artboard**: `docs/design/opendesign/herdr-parity/herdr-parity-settings-attention.html` — §02 system-channel chips + native content table. Native chrome is platform; the settings chips are the visual contract.

### S7. Sound layer

- **Today**: none — no audio anywhere in `web/`.
- **Change**: one built-in sound on notification delivery (one per delivery batch), on by default, global toggle; obeys focus suppression and workspace mute; silent-failure on autoplay/no-device.
- **States to design**: none visual beyond the S8 toggle.
- **Artboard**: `docs/design/opendesign/herdr-parity/herdr-parity-toasts.html` — §03 delivery rules (no independent visual surface; S8 owns the toggle).

### S8. Settings → Attention

- **Today**: no notification settings anywhere; settings sections pattern under `web/src/systems/settings/` (per-section page + API adapter, e.g. `window-manager-layouts-api.ts:81-90` full-config PATCH).
- **Change**: new Attention section — toasts / sound / system toggles (system shows the truthful platform state), per-workspace mute list; round-trips with `[attention]` config and CLI.
- **States to design**: defaults; system-channel unavailable/denied; mute list populated/empty; orphan-mute cleanup invisible (US-011.EC-3); first-run defaults (US-011.EC-1).
- **Artboard**: `docs/design/opendesign/herdr-parity/herdr-parity-settings-attention.html` — §01 window, §02 system-channel truth, §03 mute empty + defaults.

### S9. Settings → Shortcuts table

- **Today**: `web/src/systems/settings/components/layouts/window-manager-shortcut-table.tsx:14-50` — groups Window/Tiling/Focus/Desktops/Layout; **no Tabs group** (13 tab actions unrebindable); single-chord rows via `window-manager-shortcut-row.tsx`; recorder `use-window-manager-shortcut-recorder.ts` (Escape cancels, default-recording deletes override, conflicts `blocked`/`shadowed`).
- **Change**: Tabs group added; new actions listed (session cycling, jump-to-attention, focus-last, desktop switch/create, sidebar toggle, cheatsheet); rows render chord arrays (primary + alternates) and range families compactly; conflict diagnostics name array members and expanded range digits (US-024.EC-2, US-025.EC-3).
- **States to design**: multi-chord row; range-family row; partial range override; conflict states (blocked/shadowed) across arrays; unbound-by-default actions.
- **Artboard**: `docs/design/opendesign/herdr-parity/herdr-parity-settings-shortcuts.html` — §01 table (Tabs + arrays + ranges), §02 member-named conflicts.

### S10. Terminal preset control

- **Today**: none — no preset concept.
- **Change**: one control in the Shortcuts section: preview (old → new diff, collisions flagged incl. layout-hazard warnings), confirm applies atomically as plain overrides, revert restores the exact prior state (US-026.AC-1/AC-2, EC-1/EC-3).
- **States to design**: preview diff (with and without collisions); applied state (revert affordance); idempotent re-apply.
- **Artboard**: `docs/design/opendesign/herdr-parity/herdr-parity-settings-shortcuts.html` — §03 preview / applied / revert.

### S11. Shortcuts cheatsheet

- **Today**: `web/src/systems/os/components/os-shortcuts-dialog.tsx:30-41` — hardcoded `SHELL_ROWS` (⌘K/⌘N/⇧⌘G/Esc; ⌘[ missing), prefix-matched registry sections; opened only from menubar Help; no `?` binding; glyph rendering duplicated 3× (`window-manager-shortcuts.ts:61`, `window-manager-shortcut-row.tsx:19`, `window-manager-behavior-picks.tsx:94`).
- **Change**: `?` opens it (outside editables); every row derives from the live registry (defaults + overrides + preset); surface-local bindings (palette, permission dock 1-4, steer, composer) appear as a read-only reference section; one shared glyph helper everywhere.
- **States to design**: full sheet with arrays (primary + alternates rule — one canonical surfacing), read-only section separation, post-rebind freshness (US-028.EC-3).
- **Artboard**: `docs/design/opendesign/herdr-parity/herdr-parity-cheatsheet.html` — §01 two-column sheet, §02 freshness + shared glyph helper.

### S12. Palette nested view mechanism

- **Today**: `web/src/systems/os/hooks/use-os-command-palette.ts` (343 lines — view-model) + `os-command-palette.tsx` (cmdk `Command*` primitives); exactly one mode exists: `destinationWindowId` (palette intent) flips title/placeholder/onSelect — the seam to generalize; entries are flat `CommandItem`s in three group files.
- **Change**: generic view stack — root results can push a registered view; breadcrumb path in the input area; backspace on empty query pops; dismiss closes all; per-view search + keyboard nav identical everywhere; built-in registry only (v1).
- **States to design**: root with view entries; one level deep; three levels deep (breadcrumb truncation, US-029.EC-1); view empty state (US-029.EC-2); live refresh without selection steal (US-029.EC-4).
- **Artboard**: `docs/design/opendesign/herdr-parity/herdr-parity-palette-sessions.html` — §01 live stack, §02 depth + empty.

### S13. Palette Sessions view

- **Today**: sessions appear only as flat search results in `os-command-palette-results.tsx` (alphabetical).
- **Change**: the first registered view — state-filter chips (needs-you / working / finished / idle), attention-first order, Show all scope option (US-031.AC-2), enter focuses the session window and closes the palette; `done` arrival marks seen. Entry points: ⌘E opens the palette already pushed into this view; a root palette entry pushes it too.
- **States to design**: unfiltered; each chip active; chip with zero matches (one-keystroke clear, US-030.EC-1); Show all widened list with workspace labels; hundreds of sessions (US-030.EC-3); closed-window session activation restoring the window (US-030.EC-4).
- **Artboard**: `docs/design/opendesign/herdr-parity/herdr-parity-palette-sessions.html` — §01 Sessions view, §03 chips / zero-match / scope.

### S14. Session window header status line

- **Today**: `web/src/systems/session/components/session-status-line.tsx:13-22` — a second, conflicting badge→tone map (e.g. `running: success` vs sidebar `running: accent`).
- **Change**: consume the same exported map as S1; render the two new badges; no independent tone decisions left anywhere.
- **States to design**: covered by S1's state set (same tokens, larger presentation).
- **Artboard**: `docs/design/opendesign/herdr-parity/herdr-parity-sidebar.html` — §01 dictionary, §05 status-line scale.

## Component plan (design → production mapping)

### Rules

- Compose from `@compozy/ui` primitives (`StatusDot`, `PillDot`, `Kbd`, `KbdGroup`, `Command*`, `Popover`, `Switch`, `Badge`, `Button`) and existing domain composites; artboard CSS (`herdr.css`) is a visual contract, never a stylesheet to import.
- One exported badge→tone/glyph dictionary feeds S1, S3, S13, S14 (kills the G7 duplicate); one exported modifier-glyph helper feeds S9, S11 and the 7 hardcoded JSX chord labels. Both dictionaries follow `DESIGN-NOTES.md` locked facts — do not re-open the tone call.
- The bell keeps its presentational split: `attention-bell.tsx` renders rows; all derivation stays in `attention-model.ts` pure functions; staleness remains a first-class input (stale source contributes no count, and no notification ever fires from a stale stream).
- Visual-language authority is the named board; lossy axes (demo data, fixture copy, redrawn host chrome, hand-rolled markup) resolve toward runtime truth, `COPY.md`, `@compozy/ui`, and the live host surface — record each as an authorized difference.

### New `@compozy/ui` primitives

None. The inventory covers every visual need; the two shared dictionaries above are domain lib modules, not primitives.

### New domain components

| Component / module | System | Composed from | Used by |
| --- | --- | --- | --- |
| `SessionBadgeMark` map unification (extend `SessionStatusMark`) | `session` | `PillDot`, `StatusDot` | S1, S3, S13, S14 |
| `SessionAttentionToast` content | `os` | sonner content slot, `Kbd`-free plain copy | S5 |
| `AttentionBellSections` (rows grouped Needs you / Finished) | `os` | existing bell row + `Popover` | S3 |
| `useDocumentTitleBadge` hook | `os` | `useSyncExternalStore` pattern (mirrors `use-document-visible.ts`) | S4 |
| `useAttentionNotifier` (edge consumption → toast/system/sound dispatch) | `os` | attention stream + `user-feedback.ts` | S5, S6, S7 |
| `SettingsAttentionSection` | `settings` | `Switch`, list rows | S8 |
| `ShortcutPresetCard` (preview diff + apply/revert) | `settings` | `Button`, `Kbd`, diff rows | S10 |
| `PaletteViewStack` (breadcrumb + view registry glue) | `os` | `Command*` primitives | S12 |
| `SessionsPaletteView` | `os` | `CommandItem`, filter chips (`Badge`) | S13 |

### Signal & state mapping (locked — `DESIGN-NOTES.md`)

| State | Tone token | Glyph/shape | Notes |
| --- | --- | --- | --- |
| `waiting-for-input` | danger | question | needs-you class |
| `waiting-for-auth` | danger | shield | needs-you class; re-inked from warning |
| `failed` | danger | cross | needs-you class |
| `done` | info | check | finished-unseen; clears on focus; never inflates needs-you |
| `running` | accent | filled, pulse | unchanged |
| `idle` | success | filled | unchanged |
| `hung` | warning | filled | runtime health |
| `unhealthy` | warning | ring | runtime health |
| `stopped` | faint | solid ring | unchanged |
| `unknown` | neutral | dashed hollow ring | honesty state — distinct from `stopped` |

Needs-you shares one tone (class identity) and differs by glyph (action identity) — color never the sole channel. Precedence (US-001.EC-1): `waiting-for-auth` outranks `waiting-for-input`; `running` beats `done` on a new turn (US-002.EC-4). Two presentation scales, one meaning: 7–9px row marks · 18px `.sig` roundels.
