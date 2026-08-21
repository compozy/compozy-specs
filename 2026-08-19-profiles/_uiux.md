# UI/UX Change Map: Profiles

Every UI surface Profiles touches: where it lives today, what changes, which states must be designed, and the reference artboard each surface needs. Artboards land under `docs/design/opendesign/profiles/` and become the visual contracts the implementation tasks cite.

Companions: `_spec.md` Part I (behavior authority), `_user_stories.md` (states come from ACs/ECs), `_dx.md` (the non-UI half of the surface), `analysis/11_arch-path-map.md` (today-state anchors).

## Design constraints (apply to every artboard)

- **Identity color is user data, never a signal.** The profile's color/symbol render as chosen (Linear-style free hex + suggested palette); the signal palette keeps its meanings — needs-setup `#D6A647` warning, destructive `#E0635A` danger, success toasts `#5FBF85`, archived/muted `#8E8EB5` info. Foreground on identity color is computed for AA contrast; color never carries meaning alone (name + symbol always present).
- **Quiet until plural.** With only `default`, the switcher renders neutral and minimal (US-007.EC-2, US-010.EC-1); no profile ceremony anywhere before a second profile exists.
- **Truthful UI.** No per-profile controls for machine facts: sandboxes, scheduler budgets, native provider logins say "machine-level" plainly (US-021.EC-3); worktrees always visible with owner tag (US-009.EC-1); aggregate rows always owner-labeled; "separation, not security" copy at profile creation/management.
- **Default-read contracts (SD-012)** are listed per surface below; anything beyond them demotes to disclosure.
- **Keyboard & a11y**: switcher fully arrow-navigable; picker searchable by keyboard; dialogs trap focus; destination chip is text, not color.
- **Copy**: `COPY.md` register; the switcher carries the one-sentence boundary answer: "Work is separate per profile. Project folders and machine tools are shared." No helper prose under headings elsewhere.

## Surface map

| #   | Surface                                  | Kind   | Core change                                                        | Stories |
| --- | ---------------------------------------- | ------ | ------------------------------------------------------------------ | ------- |
| S1  | Menubar profile switcher                 | new    | Identity element + switch/create/All entry                         | US-007, US-010, US-011, US-014 |
| S2  | Creation primitives under "All"          | modify | Destination chip "→ default" + owner toast                         | US-012 |
| S3  | Work listings & live views               | modify | Server-scoped rows; owner labels in aggregate; owner banner on deep links; empty states name the profile | US-008, US-009, US-011, US-031 |
| S4  | Settings → Profiles page                 | new    | List/create/edit/archive list/delete; selection map visibility     | US-001–007, US-032 |
| S5  | Identity picker (in create/edit)         | new    | Icons/emojis tabs + search + color (Linear-style)                  | US-001, US-002 |
| S6  | Rename dialog                            | new    | Tiered rename: machine auto, repo offers, dormant placements       | US-003 |
| S7  | Archive / delete confirmations           | new    | Blocked-by-sessions, paused automations list, delete enumeration   | US-004–006 |
| S8  | Extension install & detail               | modify | Declared-profiles summary, needs-setup state, dormant placements   | US-022–024 |
| S9  | Workspace registration & workspace views | modify | "Declares content for profiles X, Y — create?" hint                | US-018, US-019 |
| S10 | Observe / usage surfaces                 | modify | Profile dimension on usage; labeled aggregate on machine dashboards | US-013 |
| S11 | Globe + workspace picker interplay       | modify | Phase-0 semantics: globe = across-workspaces view; picker list identical across profiles | US-010, US-011, US-030 |
| S12 | Window/desktop restoration                | modify | Per-profile desktops restore on switch                             | US-026 |
| S13 | Settings → notification presets           | modify | Per-profile preset enablement rows                                 | US-025 |

### S1. Menubar profile switcher

- **Today**: identity cluster is mark → globe (`GlobalScopeToggle`, `web/src/systems/os/components/global-scope-toggle.tsx:22-83`) → workspace chip (`web/src/systems/os/components/os-menubar.tsx:163-188`, cluster at `:209-226`); all props wired in `desktop-menubar.tsx:183-250`. No profile concept.
- **Change**: the profile switcher lives on the **right side of the menubar, immediately before Settings** — the conventional profile-selector position (operator ruling, surface grill Q7). The left identity cluster (mark → globe → workspace chip) is untouched. Quiet/neutral single state; after a second profile exists it shows glyph+name. Menu: profiles with glyph/name/active mark, "All profiles" state, "Create profile…", the one-sentence boundary answer, archived count line linking to Settings.
- **States to design**: quiet single (US-007.EC-2); plural list + active (US-010.AC-1); All state on (US-011.AC-1); switch updates remembered choice (US-010.AC-3); archived-mid-open fallback (US-010.EC-3); per-workspace restore on workspace change (US-014.AC-1); Global-lens slot (US-014.AC-3).
- **Default read (SD-012)**: answers "which context am I in? · switch to which? · am I seeing everything? · create new". Demotes: archive management, selection map, identity editing → Settings.
- **Artboard**: `profiles-switcher.html`.

### S2. Creation primitives under "All"

- **Today**: shared creation entry points (new session, task, loop, automation) carry no destination concept.
- **Change**: while the All state is on, every shared creation primitive shows a fixed text chip "→ default" before commit and an owner-naming success toast after (label only, never a picker — ADR-005).
- **States to design**: chip visible pre-commit (US-012.AC-1); toast naming owner (US-012.AC-2); palette/deep-link creation parity (US-012.EC-2).
- **Default read**: the chip is part of the default read of creation surfaces — never a tooltip.
- **Artboard**: `profiles-aggregate.html` (shared with S3).

### S3. Work listings & live views

- **Today**: session catalog stream is flat and client-filtered (`web/src/systems/session/hooks/use-session-catalog-streams.ts:40-65`; reconnect generations at `session-catalog-streams-store.ts:34-50`); rows have no owner concept; query keys are workspace-qualified (`web/src/systems/workspace/lib/query-keys.ts:1-19`).
- **Change**: all listings consume server-scoped reads (no client filtering); profile switch bumps stream generation and query-key namespace; aggregate mode adds an owner tag per row (glyph+name, archived owners muted); deep link to another profile's item renders an owner banner with one-tap switch — backed by the labeled aggregate-by-id read (`?all_profiles=true` on the single get), never a client-side exception; empty states name the active profile.
- **States to design**: scoped list (US-009.AC-1); aggregate labeled rows incl. archived owner (US-011.AC-1, US-004.AC-2); owner banner (US-009.EC-2); empty state "No sessions in Marketing yet" (US-009.EC-3); worktree rows always tagged (US-009.EC-1).
- **Default read**: unchanged questions per listing; the owner tag appears only in aggregate mode (scoped views stay tag-free — calm default).
- **Artboard**: `profiles-aggregate.html`.

### S4. Settings → Profiles page

- **Today**: no page; Settings follows the route-partial convention (`web/src/routes/_app/settings/` + `-*-page.tsx`).
- **Change**: new Profiles page: active list (glyph, name, work count), create, edit identity, rename, archive; archived list under disclosure with unarchive/delete-when-empty; the workspace→profile selection map inspectable; "separation, not security" line lives here.
- **States to design**: list w/ single default (US-007); create dialog incl. validation errors (US-001.EC-1..3); archived list + unarchive (US-005); delete enumeration (US-006.AC-3); remote surfaces render management absent, not disabled (US-032.AC-2 — truthful UI).
- **Default read**: answers "which profiles exist? · create one · which is active where?". Demotes: archived list, selection map, identity editing.
- **Artboard**: `profiles-settings.html`.

### S5. Identity picker

- **Today**: nothing comparable in `@compozy/ui` (`packages/ui/src/exports/foundation.ts:27-35` has Avatar only; no color/icon picker anywhere in the barrels).
- **Change**: Linear-style picker (operator's reference): tabs Icons | Emojis, search field, icon grid tinted by chosen color, color row (suggested palette + free hex input).
- **States to design**: icons tab, emojis tab, search-empty, custom hex invalid (US-002.EC-1), auto-assigned starter (US-001.AC-3).
- **Artboard**: `profiles-picker.html`.

### S6. Rename dialog

- **Change**: one dialog: new-name field; result plan grouped in tiers — machine folders (automatic, informational), repo folders per workspace (pre-checked offers → pending changes to commit, US-003.AC-3), extension placements (informational: will sleep, US-003.AC-4); unreachable workspaces listed as skipped (US-003.EC-3).
- **States to design**: no repo matches; matches with mixed accept/decline; name-taken inline error (US-003.EC-1); permanent-profile refusal for `default` (US-003.EC-2).
- **Artboard**: `profiles-lifecycle.html`.

### S7. Archive / delete confirmations

- **Change**: archive confirm lists paused automations (US-004.AC-4 flow, ADR-001); blocked state names running sessions (US-004.EC-1); unarchive lists automations for explicit reactivation (US-005.AC-2); delete confirm enumerates removals (US-006.AC-3) or routes to archive when work exists (US-006.AC-2).
- **States to design**: each of the four dialogs + their refusal states; danger styling only on the destructive action.
- **Artboard**: `profiles-lifecycle.html`.

### S8. Extension install & detail

- **Today**: install confirmation and extension detail exist; no profile concepts.
- **Change**: install/update summary lists declared profiles ("will create profile growth — needs 1 credential", US-023.AC-2); detail page shows placement per resource (machine-wide vs profile name), dormant placements with create hint (US-022.EC-1), per-profile enablement control (US-024), needs-setup signal on created profiles until credential asks are filled (US-023.AC-3).
- **States to design**: pre-install summary; post-install needs-setup; dormant placement hint; enablement toggle per profile; uninstall copy stating the profile stays (US-023.EC-4).
- **Default read**: detail answers "what does it add, where, is it on here?"; placement matrix demotes to disclosure.
- **Artboard**: `profiles-extension.html`.

### S9. Workspace registration & workspace views

- **Change**: registering (or opening) a workspace whose repo declares profile folders surfaces the hint "this project declares content for profiles dev, marketing — create?" with one action per name (US-019.AC-1..2); hint drops adopted names; renamed repo folders re-point the hint (US-019.EC-2).
- **States to design**: hint with 1..N names; partially adopted; ignored (dormant, no nagging).
- **Artboard**: `profiles-hints.html`.

### S10. Observe / usage surfaces

- **Today**: per-session cost renders in the session inspector only (`web/src/systems/session/components/session-inspector-sections.tsx`); machine dashboards are unscoped routes.
- **Change**: scoped surfaces inherit profile scoping automatically (nothing new to render); machine-wide observe/usage dashboards adopt the labeled-aggregate read with a per-profile breakdown row (US-013.AC-2). No new dashboard is invented.
- **States to design**: aggregate usage with archived owners included (US-013.EC-1); pre-profiles history under `default` (US-013.EC-2).
- **Artboard**: `profiles-aggregate.html` (shared).

### S11. Globe + workspace picker interplay (phase 0)

- **Today**: globe toggles Global-as-pseudo-workspace (`global-scope-toggle.tsx:22-83`); workspace chip reads project or Global (`os-menubar.tsx:163-188`); `~/` appears as a workspace.
- **Change**: globe becomes the across-workspaces view (aggregate; creations = no-workspace work, US-030.AC-2); `~/` disappears from workspace lists (US-030.EC-1); workspace list renders identically in every profile (US-010.AC-4); profile × globe × workspace axes compose visibly (US-011.AC-2).
- **States to design**: globe-on with profile scoped; globe-on with All; zero-workspaces fresh start (US-030.EC-3).
- **Artboard**: `profiles-switcher.html` (shared).

### S12. Window/desktop restoration

- **Today**: one window-manager snapshot per workspace via client state (`internal/daemon/window_manager_repository.go:17-28`); client view state in `window-manager-store.ts:122-160`.
- **Change**: switching profiles restores that profile's desktops/windows per workspace (US-026.AC-1); new profile starts clean (US-026.AC-2); no leakage either way (US-026.EC-2).
- **States to design**: switch-restore transition; first-entry clean state.
- **Artboard**: none — behavior only, no new visual language; covered by switcher artboard notes.

### S13. Notification preset enablement

- **Today**: preset library is machine-wide; no per-profile dimension on the notifications/attention Settings surface.
- **Change**: that Settings surface gains per-profile enablement — rows show the effective state for the active profile and changes act on it, backed by the preset-enablement routes (`_dx.md`), with CLI parity.
- **States to design**: fresh profile default-on (US-025.AC-2); disabled row; archived-owner deliveries paused note (US-025.EC-1).
- **Artboard**: `profiles-settings.html` (shared).

## Component plan (design → production mapping)

### Rules

- Compose from `@compozy/ui` first; artboard CSS is a visual contract, never a stylesheet to import. Identity color flows as data (CSS custom property per row/chip), never as new tokens.
- The switcher composes `CommandSelect*` (`packages/ui/src/exports/listing.ts:53-60`) + `Avatar` family (`foundation.ts:27-35`); it does not fork `GlobalScopeToggle`.
- Profile-scoped query keys copy the workspace-qualified pattern (`query-keys.ts:1-19`); stream consumers bump generation on profile switch (`session-catalog-streams-store.ts:34-50`).

### New `@compozy/ui` primitives

- **`SymbolPicker`** — tabs (icons/emojis) + search + grid + color row (palette + free value). Justified: no picker primitive exists in any barrel; generic beyond profiles (future team/agent identity). Ships with story + test.
- None else — everything further is composition.

### New domain components (`web/src/systems/profiles/`)

- `ProfileGlyph` — Avatar composed with identity color + icon/emoji; used by S1, S3, S4, S8.
- `ProfileSwitcher` — CommandSelect composition with All state + create entry; used by S1.
- `ProfileOwnerTag` — compact glyph+name row tag for aggregate mode; used by S3, S10.
- `ProfileDestinationChip` — fixed "→ default" text chip; used by S2.
- `ProfileCreateDialog` / `ProfileRenameDialog` / `ProfileArchiveDialog` / `ProfileDeleteDialog` — S4–S7 flows.
- `ProfilesSettingsPage` partial (`-profiles-settings-page.tsx`) — S4.
- `ExtensionDeclaredProfiles` section — S8.
- `WorkspaceProfilesHint` — S9.

### Signal & state mapping

| Design state | Primitive + token |
| --- | --- |
| Active profile marker | ring in identity color on `ProfileGlyph` (data, not signal) |
| Needs setup (credential asks) | warning `#D6A647` badge on glyph/row |
| Archived owner in aggregate | info `#8E8EB5`, muted row tag |
| Destructive delete action | danger `#E0635A` on the confirm action only |
| Created/renamed success toast | success `#5FBF85` toast accent |
| Dormant content hint | info `#8E8EB5` inline hint with create action |
