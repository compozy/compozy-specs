# User Stories: Profiles

Canonical behavior catalog for the Profiles feature. Companion to `_spec.md`; consumed by `_spec.md` Part II (component mapping), `_uiux.md` (surface states), and `_tests.md` (coverage matrix). Binding inputs: `DECISIONS.md` D1–D27 (as amended 2026-08-20) and `adrs/adr-001..003.md`.

## Personas

- **Operator** — the person running CompozyOS on their own machine, separating their own working contexts (product dev, marketing, finance back-office). Technical today; the data model is normie-ready but v1 makes no non-technical claim.
- **Team member** — another operator who shares a repository that carries committed per-profile folders. They consume the team's division of contexts on their own machine; they never manage anyone else's daemon.
- **Extension author** — builds and ships an extension that contributes resources machine-wide, places resources into named profiles, and may declare profiles that are created automatically at install.
- **Agent** — an autonomous session, loop, or automation running inside the daemon. It inherits a profile at creation, operates the feature through structured command surfaces, and may converse with agents in other profiles.

## Story Index

| ID     | Feature Area           | Persona          | Story                                                        |
| ------ | ---------------------- | ---------------- | ------------------------------------------------------------ |
| US-001 | Identity & lifecycle   | Operator         | Create a profile with name, color, and icon/emoji            |
| US-002 | Identity & lifecycle   | Operator         | Edit a profile's identity at any time                        |
| US-003 | Identity & lifecycle   | Operator         | Rename a profile with tiered folder handling                 |
| US-004 | Identity & lifecycle   | Operator         | Archive a profile that still owns work                       |
| US-005 | Identity & lifecycle   | Operator         | Unarchive a profile                                          |
| US-006 | Identity & lifecycle   | Operator         | Delete an empty profile                                      |
| US-007 | Identity & lifecycle   | Operator         | Rely on a permanent `default` profile                        |
| US-008 | Scoped work            | Operator         | New work lands in the active profile                         |
| US-009 | Scoped work            | Operator         | Listings and live views show only the active profile's work  |
| US-010 | Scoped work            | Operator         | Switch the active profile                                    |
| US-011 | Scoped work            | Operator         | See everything at once in "All profiles"                     |
| US-012 | Scoped work            | Operator         | Create while in "All profiles" and land in `default`         |
| US-013 | Scoped work            | Operator         | See usage and spend attributed per profile                   |
| US-014 | Selection & return     | Operator         | Return to a workspace and get its remembered profile         |
| US-015 | Selection & return     | Operator / Agent | Select a profile per command in the terminal                 |
| US-016 | Resource layers        | Operator         | User-level resources appear in every profile                 |
| US-017 | Resource layers        | Operator         | Keep personal resources per profile                          |
| US-018 | Resource layers        | Operator         | Commit a shared base and per-profile folders to a repo       |
| US-019 | Resource layers        | Team member      | Adopt a team's committed profile division                    |
| US-020 | Config & credentials   | Operator         | Override configuration per profile                           |
| US-021 | Config & credentials   | Operator         | Override provider credentials per profile                    |
| US-022 | Extensions             | Extension author | Place extension resources into named profiles               |
| US-023 | Extensions             | Extension author | Declare profiles that are created automatically at install   |
| US-024 | Extensions             | Operator         | Enable or disable an extension per profile                   |
| US-025 | Extensions             | Operator         | Enable notification presets per profile                      |
| US-026 | Per-profile state      | Operator         | Keep separate desktops and window layouts per profile        |
| US-027 | Per-profile state      | Agent            | Recall profile-scoped memory                                 |
| US-028 | Agents & automation    | Agent            | Run under an immutable profile binding                       |
| US-029 | Agents & automation    | Agent            | Converse across profiles on the shared network               |
| US-030 | Foundation fixes       | Operator         | Treat Global as a view, not a workspace                      |
| US-031 | Foundation fixes       | Operator         | Receive only in-scope session updates                        |
| US-032 | Manageability & remote | Operator         | Manage profiles only from the machine                        |
| US-033 | Manageability & remote | Agent            | Inspect profiles and the active selection                    |

## Identity & lifecycle

### US-001: Create a profile with name, color, and icon/emoji

**As an** operator, **I want** to create a named profile with a color and an icon or emoji, **so that** I get a visually distinct working context I can switch into.

Acceptance criteria:

- AC-1: Given the switcher or the Settings profiles page, when I create a profile with a valid name, then it appears in the switcher with its color and symbol and becomes the active profile immediately.
- AC-2: Given the creation form, when I open the symbol picker, then I can search and choose an icon from the product icon set or an emoji, and pick a color from a suggested palette or a free color value.
- AC-3: Given I skip identity choices, when I create with only a name, then a starter color and symbol are auto-assigned and editable later.
- AC-4: Given any creation surface, when creation succeeds, then the same profile is visible to the terminal and programmatic surfaces with identical identity fields.
- AC-5: Given the terminal or programmatic surfaces, when I create a profile there, then the same validation and result apply, with structured output and named, deterministic errors (invalid name, name in use, reserved name).

Edge cases:

- EC-1: Name with spaces, uppercase, or symbols outside the allowed set → rejected with a plain message stating the allowed shape (short, lowercase, letters/digits/hyphens, starts with a letter).
- EC-2: Name already in use by an active or archived profile → rejected with a message naming the holder ("'dev' exists — archived; unarchive it or pick another name").
- EC-3: Reserved names (`default`, `all`, `global`) → rejected with the reserved-name reason.
- EC-4: Creating while "All profiles" is on → the profile is created normally, becomes active, and the aggregate state turns off.
- EC-5: Two creation requests with the same name racing → exactly one wins; the second receives the name-in-use rejection.
- EC-6: Creation succeeds but a later step of the create flow fails (e.g., identity save) → the profile exists with defaults; repair happens by editing, never by a half-created invisible state.

### US-002: Edit a profile's identity at any time

**As an** operator, **I want** to change a profile's color, icon, or emoji whenever I want, **so that** the visual identity stays useful as contexts evolve.

Acceptance criteria:

- AC-1: Given the profile's settings, when I change color or symbol, then the switcher, labels, and every surface showing the profile update without restart.
- AC-2: Given the `default` profile, when I edit its color or symbol, then the change applies (cosmetics are allowed on `default`).

Edge cases:

- EC-1: Invalid color value → rejected inline; previous color stays.
- EC-2: Identity edits on an archived profile → allowed from the archived list (it may return later).
- EC-3: Two surfaces editing identity concurrently → last write wins; both surfaces converge on the saved value.

### US-003: Rename a profile with tiered folder handling

**As an** operator, **I want** to rename a profile and have the system handle every folder tied to the old name, **so that** a rename never silently breaks my resources.

Acceptance criteria:

- AC-1: Given a profile with work and resources, when I rename it, then all existing work stays attached (history, attribution, and listings unchanged) — the label changes, nothing moves.
- AC-2: Given machine-side personal folders for the profile, when the rename completes, then those folders are renamed automatically with no operator action.
- AC-3: Given registered workspaces containing committed folders matching the old name, when I rename, then the flow lists them and offers to rename each (pre-checked); accepted ones become pending changes in those projects for me to commit.
- AC-4: Given extension placements naming the old name, when the rename completes, then those placements are reported as now-dormant with a hint (the extension's files are never edited).
- AC-5: Given the terminal or programmatic surfaces, when I rename there, then the same tiered handling applies with structured output — machine folders renamed, repo offers listed and answerable, dormant placements named — and named errors (name in use, permanent profile).

Edge cases:

- EC-1: Renaming to a name held by another profile (active or archived) → rejected with the name-in-use message.
- EC-2: Renaming `default` → rejected (permanent identity).
- EC-3: A workspace with a matching committed folder is unreachable at rename time → listed as skipped; its content goes dormant with the standard hint until renamed or the name returns.
- EC-4: I decline the repo folder rename for one project → that project's content goes dormant with the hint; accepting later is a normal folder rename in the project.
- EC-5: Rename while a session in that profile is running → allowed; running work keeps functioning and displays the new name.
- EC-6: Renaming back to a previously used name → dormant content bound to that name wakes up.

### US-004: Archive a profile that still owns work

**As an** operator, **I want** to archive a profile I no longer use, **so that** my switcher stays calm without losing any history.

Acceptance criteria:

- AC-1: Given a profile that owns work, when I archive it, then it disappears from the switcher and every day-to-day selection surface.
- AC-2: Given an archived profile, when I open "All profiles", then its work still appears, owner-labeled like everything else.
- AC-3: Given remembered workspace selections pointing at it, when it is archived, then those workspaces fall back to `default` on next entry.
- AC-4: Given the delete action on a profile with work, when I invoke it, then I am offered archive instead, with plain copy explaining why.
- AC-5: Given the terminal or programmatic surfaces, when I archive there, then the result and refusals are identical (running-sessions block, `default` permanence), with structured output and named errors.

Edge cases:

- EC-1: Archiving while one of its sessions is running → blocked with a message naming the running work; stop it first (archiving mid-run would hide live work from every scoped view).
- EC-2: Archiving `default` → rejected (permanent).
- EC-3: Work arriving for an archived profile through a still-scheduled automation → the automation is paused at archive time and listed in the archive confirmation; nothing new is created for an archived profile.
- EC-4: Archive requested twice concurrently → idempotent; second request reports already archived.
- EC-5: Extension-declared profile archived → never auto-recreated (creation marker holds; ADR-002).
- EC-6: Queued (not yet claimed) runs owned by the profile at archive time → frozen: never picked up while archived (the claim step itself checks the owner's state), counted in the archive confirmation; unarchive makes them claimable again.

### US-005: Unarchive a profile

**As an** operator, **I want** to restore an archived profile, **so that** a paused context can come back exactly as it was.

Acceptance criteria:

- AC-1: Given the archived list in Settings, when I unarchive a profile, then it returns to the switcher with identity, work, resources, config, and credential overrides intact.
- AC-2: Given automations paused by the archive, when I unarchive, then they are listed for explicit reactivation (not silently resumed).

Edge cases:

- EC-1: Unarchiving when an active profile now holds its old name → impossible by construction (archived profiles keep their names reserved; see US-001.EC-2).
- EC-2: Unarchive twice concurrently → idempotent.

### US-006: Delete an empty profile

**As an** operator, **I want** to permanently delete a profile that owns no work, **so that** mistakes and experiments can be fully removed.

Acceptance criteria:

- AC-1: Given a profile with zero owned work, when I delete it, then it is removed everywhere, its personal folders are removed, and its name frees immediately.
- AC-2: Given a deletion attempt on a profile that owns work, when I confirm the offered alternative, then the flow archives instead (US-004.AC-4).
- AC-3: Given the profile still holds personal resources, configuration, or credential overrides, when I confirm deletion, then the confirmation enumerates what will be removed — resources by kind and count, config layers, credential overrides — before I commit.
- AC-4: Given the terminal or programmatic surfaces, when I delete there, then the same gate and enumeration apply, with structured output and named errors (owns work, permanent profile).

Edge cases:

- EC-1: Deleting `default` → rejected (permanent).
- EC-2: Delete racing with new work creation in that profile → whichever commits first wins; delete then fails with "owns work" and offers archive.
- EC-3: Deleted extension-declared profile → never auto-recreated; reinstalling the same extension instance does not recreate it (marker persists; ADR-002).
- EC-4: Profile had credential overrides → they are removed with it; nothing falls back silently to machine credentials mid-run (no runs exist — the profile was empty).
- EC-5: Remembered workspace selections pointing at the deleted profile → swept; affected workspaces fall back to `default` on next entry.

### US-007: Rely on a permanent `default` profile

**As an** operator, **I want** everything to belong to `default` when I never chose a profile, **so that** the feature is invisible until I need it.

Acceptance criteria:

- AC-1: Given a fresh install, when I use CompozyOS without touching profiles, then every behavior matches today's product; the switcher stays quiet/neutral.
- AC-2: Given any surface with no profile selected, when work is created, then it belongs to `default` and says so where owner labels appear.
- AC-3: Given the `default` profile, when I try to archive, delete, or rename it, then the action is rejected with the permanence reason.

Edge cases:

- EC-1: All other profiles archived → the product degrades gracefully to single-profile behavior in `default`.
- EC-2: First profile ever created → the switcher becomes a visible identity element from that moment (quiet before, present after).

## Scoped work

### US-008: New work lands in the active profile

**As an** operator, **I want** every session, task, loop run, and automation I start to belong to the profile I'm in, **so that** contexts never mix at creation time.

Acceptance criteria:

- AC-1: Given active profile `marketing`, when I start a session / enqueue a task / run a loop / create an automation, then the new item is owned by `marketing` and shows that owner wherever owner labels appear.
- AC-2: Given a child item produced by owned work (a spawned session, a retry, a review round, a loop step), when it is created, then it belongs to its parent's profile with no way to diverge.
- AC-3: Given work created through any surface (web, terminal, programmatic, automation trigger), when it is created, then the same ownership rule applies uniformly.

Edge cases:

- EC-1: Automation fires while its owning profile is archived → nothing is created (US-004.EC-3); the pause reason is inspectable.
- EC-2: Two terminals with different profile selections create work simultaneously → each item belongs to its own terminal's profile; no cross-talk.
- EC-3: Work created by an agent inside a session → belongs to the session's profile regardless of any flag the agent passes (agents cannot re-aim ownership; ADR-001 context, D9).
- EC-4: Work arrives unattended through a bridge → owned by the bridge instance's profile (one entry point, one owner; provenance-based routing is future scope, ADR-011).

### US-009: Listings and live views show only the active profile's work

**As an** operator, **I want** every list, counter, and live update to show only my active profile's work, **so that** switching profiles really changes what I see.

Acceptance criteria:

- AC-1: Given active profile `dev`, when I open sessions, tasks, loops, automations, or attention/approval surfaces, then only `dev`-owned items appear — including counts and badges.
- AC-2: Given a live surface already open, when new work arrives for another profile, then nothing appears in my view (the daemon filters before sending; nothing is filtered client-side).
- AC-3: Given the filter fails or scope is indeterminate, when a view loads, then it shows nothing rather than everything (fail-closed).

Edge cases:

- EC-1: Worktrees are the ruled exception: visible in every profile, always owner-tagged (hiding is future capability gating).
- EC-2: An item's deep link opened while another profile is active → the item renders with an owner banner and a one-tap switch to its profile; surrounding listings stay scoped to the active profile.
- EC-3: Zero items in the active profile → the empty state names the active profile ("No sessions in Marketing yet"), so an unexpected empty view explains itself.
- EC-4: 100× typical volume in one profile → other profiles' views are unaffected (scoping bounds every listing's population).
- EC-5: A link to a loop output from another profile's run → unreachable in the active scope; outputs are reachable only through a run the current scope can see, and no surface lists outputs outside a run.

### US-010: Switch the active profile

**As an** operator, **I want** to switch profiles from the topbar identity element, **so that** changing context is one gesture.

Acceptance criteria:

- AC-1: Given profiles exist, when I open the switcher, then I see each profile with color+symbol+name and the active one marked; choosing one refilters every open surface to it.
- AC-2: Given the switcher, when I look for guidance, then it answers "what stays together, what separates" in one plain sentence (work separates by profile; project folders and machine resources are shared).
- AC-3: Given a switch, when it completes, then the remembered choice for the current workspace (or the Global lens) updates (ADR-003).
- AC-4: Given any active profile, when I open the workspace picker, then the workspace list is identical to every other profile's — workspaces are machine-global; only work, resources, and settings change with the profile.

Edge cases:

- EC-1: Only `default` exists → the switcher is quiet/neutral (no list ceremony for one entry).
- EC-2: Switch while a long-running session streams → the stream belongs to the previous profile's scope and leaves the view; the session keeps running unaffected.
- EC-3: Profile archived from another surface while its switcher is open → choosing it reports it just became unavailable and falls back to `default`.
- EC-4: Two clients open with different active profiles → each keeps its own view (selection is per-client state); only the remembered choice of the lens being switched updates.

### US-011: See everything at once in "All profiles"

**As an** operator, **I want** an aggregate state that shows every profile's work labeled by owner, **so that** I can audit the whole machine without switching one by one.

Acceptance criteria:

- AC-1: Given the switcher, when I choose "All profiles", then work listings show every profile's items (archived owners included), each row owner-labeled.
- AC-2: Given the aggregate state, when I combine it with the Global toggle or a specific workspace, then the axes compose (e.g., All profiles × workspace X = everyone's work in that folder).
- AC-3: Given the aggregate state, when I leave and return to a workspace later, then the remembered choice is the last real profile, never the aggregate (ADR-003).

Edge cases:

- EC-1: Aggregate with only `default` populated → looks like the scoped view plus owner labels; no special casing.
- EC-2: Terminal listings → an explicit all-profiles option on listing commands mirrors the same labeled aggregate; it is never the default.
- EC-3: Machine-wide observe/usage dashboards → use the same labeled aggregate mode (no third read mode exists).

### US-012: Create while in "All profiles" and land in `default`

**As an** operator, **I want** creation to still work in the aggregate state by filing to `default`, **so that** the aggregate never becomes a read-only maze.

Acceptance criteria:

- AC-1: Given "All profiles" is on, when I open any creation surface, then a fixed destination chip reading "→ default" is visible before I commit.
- AC-2: Given I create the item, when it succeeds, then the confirmation names the owner ("created in Default"), and the item appears owner-labeled in the aggregate view.

Edge cases:

- EC-1: I meant it for another profile → recovery is: switch to that profile and create again (moving work is future scope); the toast's owner naming is what surfaces the mistake immediately.
- EC-2: Creation from a deep link or palette while aggregate is on → same chip and toast at the shared creation primitives; no surface is exempt.

### US-013: See usage and spend attributed per profile

**As an** operator, **I want** usage and cost views broken down by profile, **so that** each context's consumption is honest — including when profiles share machine credentials.

Acceptance criteria:

- AC-1: Given usage/cost surfaces, when a profile is active, then figures cover that profile's work only.
- AC-2: Given the aggregate state, when I view usage, then per-profile figures appear labeled side by side.
- AC-3: Given a profile with its own provider key and another using the user key, when both run work, then attribution follows the owning profile regardless of which key was used.

Edge cases:

- EC-1: Work owned by an archived profile → still counted under that profile's label in aggregate views (history stays truthful).
- EC-2: Usage recorded before Profiles existed → belongs to `default` (the uniform no-choice owner).

## Selection & return

### US-014: Return to a workspace and get its remembered profile

**As an** operator, **I want** each workspace to reopen in the profile I last used there, **so that** context follows the project.

Acceptance criteria:

- AC-1: Given I used `dev` in workspace A and `marketing` in workspace B, when I switch between A and B, then the active profile follows automatically and the switcher identity updates.
- AC-2: Given a workspace I never visited, when I enter it, then the active profile is `default`.
- AC-3: Given the Global lens (no workspace), when I return to it, then its own remembered slot applies.

Edge cases:

- EC-1: Remembered profile was archived → fall back to `default` silently, with the switcher showing `default` (ADR-003).
- EC-2: Workspace unregistered and re-registered → its memory starts fresh at `default`.
- EC-3: The aggregate state was on when I left → return restores the last real profile, not the aggregate (US-011.AC-3).

### US-015: Select a profile per command in the terminal

**As an** operator or agent, **I want** an explicit per-command profile selection with a fixed precedence, **so that** scripts and parallel terminals are deterministic.

Acceptance criteria:

- AC-1: Given the root `--profile` flag, when set, then it wins over everything for that command.
- AC-2: Given `COMPOZY_PROFILE` in the environment, when no flag is set, then it wins over remembered choices.
- AC-3: Given neither, when a command resolves its workspace, then that workspace's remembered profile applies; with none, `default`.
- AC-4: Given listing commands, when I pass the all-profiles option, then rows come from every profile, owner-labeled in structured output.
- AC-5: Given machine-scoped commands (daemon start, doctor, update), when a profile selection exists anywhere, then they ignore it.

Edge cases:

- EC-1: `--profile` naming a missing or archived profile → deterministic error naming the profile and its state; nothing falls back silently.
- EC-2: Two terminals with different `COMPOZY_PROFILE` values → both operate concurrently in their own profiles without touching remembered choices.
- EC-3: Reserved words as selection values (`all`) → rejected on acting commands; aggregate reads use the dedicated listing option, never a profile name.
- EC-4: Structured outputs (tables and machine-readable forms) always include the owner field when the aggregate option is on.

## Resource layers

### US-016: User-level resources appear in every profile

**As an** operator, **I want** resources installed at the user level to be visible in every profile and workspace, **so that** my universal tools never need copying.

Acceptance criteria:

- AC-1: Given an agent/skill/loop in the user layer, when I switch to any profile or workspace, then it is available.
- AC-2: Given the fixed prerequisite (US-030), when I view any workspace, then user resources appear there too — not only under the Global view.

Edge cases:

- EC-1: User-layer resource name collides with a more specific layer → the most specific wins in that context; the shadowing is inspectable (which layer won and why).
- EC-2: No user-layer resources installed → layers below still compose normally.

### US-017: Keep personal resources per profile

**As an** operator, **I want** personal resource folders scoped to a profile, **so that** each context carries its own toolkit without leaking.

Acceptance criteria:

- AC-1: Given a resource in my `marketing` profile layer, when `marketing` is active, then it is available in every workspace; when another profile is active, it is absent.
- AC-2: Given the same resource name in the user layer and the profile layer, when the profile is active, then the profile one wins and the shadowing is inspectable.

Edge cases:

- EC-1: Operator points a personal resource location inside a registered project → rejected with the reason (personal material stays on the user side, outside the repo; repo-side folders are committed team decisions).
- EC-2: Profile renamed → personal folders follow automatically (US-003.AC-2).
- EC-3: Resource added on disk while surfaces are open → appears on the next catalog refresh in the owning scope only.

### US-018: Commit a shared base and per-profile folders to a repo

**As an** operator, **I want** to commit a project-wide resource base plus per-profile-name folders in the project, **so that** my division of contexts travels with the repository.

Acceptance criteria:

- AC-1: Given resources in the project's shared base folder, when any profile is active in that workspace, then they are available (git physics: all profiles see committed base).
- AC-2: Given resources in the project's `profiles/dev/` folder, when a profile named `dev` is active there, then they are available; under other profiles they are not.
- AC-3: Given both base and per-name folders define the same resource name, when the name matches the active profile, then the per-name one wins (most specific).

Edge cases:

- EC-1: Committed folder names a profile I don't have → content stays dormant; the workspace surface hints "this project declares content for profile 'dev' — create it?".
- EC-2: Deep nesting or unexpected files inside the profile folders → same validation rules as existing resource folders; invalid entries are reported, valid ones load.
- EC-3: Branch switch changes the committed folders → the catalog follows the working tree on refresh; no stale ghosts.

### US-019: Adopt a team's committed profile division

**As a** team member, **I want** cloning a repo with committed profile folders to light up on my machine by creating the matching profiles, **so that** the team's division works without reading docs.

Acceptance criteria:

- AC-1: Given a cloned repo with `profiles/dev/` and `profiles/marketing/` content, when I register it as a workspace, then I see the hint naming both profiles with a create action.
- AC-2: Given I create `dev` from the hint, when it completes, then the repo's `dev` resources bind immediately and the hint drops that name.
- AC-3: Given my own machine's profiles are unrelated to the team's names, when I ignore the hint, then nothing is created and nothing leaks — the content just stays dormant.

Edge cases:

- EC-1: I already had a `dev` profile of my own → the repo's `dev` content binds to it by name (that is the sharing mechanism); no seeding or mutation of my profile happens.
- EC-2: Repo renames its folder later (`dev` → `eng`) → my `dev` binding goes dormant for that repo; the hint now names `eng`.
- EC-3: Two registered repos both declare `dev` → one profile `dev` binds both repos' content, each only inside its own workspace.

## Config & credentials

### US-020: Override configuration per profile

**As an** operator, **I want** profile-level configuration layers on the machine and inside each project, **so that** each context can tune behavior without forking machine settings.

Acceptance criteria:

- AC-1: Given the five-layer config precedence (built-ins → user → user profile → project → project profile), when the same key is set in several layers, then the most specific wins and the effective value is inspectable with its winning layer.
- AC-2: Given an active profile, when I change a setting through the terminal or the settings UI, then the write targets the layer that owns the current context (the user file under `default`, the profile layer otherwise), and an explicit scope option can target user/workspace instead.
- AC-3: Given a write to a layer that is currently overridden by a more specific one, when it saves, then the response says "saved, but not applied — <layer> wins".
- AC-4: Given machine-identity keys (ports, socket, storage locations, logs, gateway/shell/marketplace), when a profile layer tries to set them, then the write is rejected with the denylist reason (a profile can never repoint machine state).

Edge cases:

- EC-1: Persona defaults (default agent/provider/sandbox) moved out of the general section → settable per profile; machine bindings remain machine-only.
- EC-2: Profile config file present for a profile that doesn't exist (hand-created) → ignored with a diagnostic; nothing half-applies.
- EC-3: Concurrent writes to the same key in different layers → both persist in their layers; effective value follows precedence; no lost updates.
- EC-4: A hook defined in a profile layer runs only while that profile is active, and the layer rules apply to hooks like any resource (most specific wins); sandbox definitions stay user-level only, never profile-layered — a profile layer declaring one is rejected with the reason.
- EC-5: After config writes in two different profiles, when the operator views the change history, then one machine timeline shows both, each entry naming the layer it changed — "what changed in this profile" stays answerable without splitting the history.

### US-021: Override provider credentials per profile

**As an** operator, **I want** a profile to optionally carry its own provider keys, **so that** billing can be separated per context while everything else defaults to the user credentials.

Acceptance criteria:

- AC-1: Given no override, when any profile runs work, then the user credentials apply (default unchanged).
- AC-2: Given `marketing` overrides one provider's key in the vault, when `marketing` work uses that provider, then the override is used; other providers and other profiles keep the user credentials.
- AC-3: Given an override exists, when work in another profile runs, then that override is unreachable from there (scoped resolution).
- AC-4: Given spend/usage views, when overrides are in play, then attribution still follows the owning profile (US-013.AC-3).

Edge cases:

- EC-1: Per-profile secrets are only accepted as vault references — process-environment references are rejected for profile scope with a plain reason (environment is machine-wide by nature).
- EC-2: Override removed while owned work is running → a warning names the user-credential fallback before confirming; in-flight work completes on the credential it resolved; the next run resolves the user default.
- EC-3: Native provider CLI logins remain single and shared across profiles in v1 → surfaces state this plainly where credentials are configured (no false promise of login separation).
- EC-4: Archived profile with overrides → overrides stay stored, unusable until unarchive (nothing new runs; US-004).

## Extensions

### US-022: Place extension resources into named profiles

**As an** extension author, **I want** to mark each contributed resource as shared (every profile) or bound to a profile name, **so that** one extension can serve several contexts precisely.

Acceptance criteria:

- AC-1: Given a resource with no placement, when the extension is enabled in a profile, then the resource is available in that profile (unplaced = every profile).
- AC-2: Given a resource placed in profile `x`, when a profile named `x` is active and the extension is enabled there, then the resource is available; in other profiles it is absent.
- AC-3: Given the operator's canonical example (1 machine-wide skill + 2 skills in `x` + 1 in `y`), when the extension is installed and profiles exist, then each skill appears exactly where declared.

Edge cases:

- EC-1: Placement names a profile that doesn't exist and isn't declared for auto-creation → dormant with the standard hint on the extension detail.
- EC-2: Effective visibility is always enablement AND placement — disabling the extension in `x` hides even correctly-placed resources there.
- EC-3: Extension update changes a resource's placement → the new placement applies from the update on; no copies linger.

### US-023: Declare profiles that are created automatically at install

**As an** extension author, **I want** to declare the profiles my extension introduces so installing it creates them properly, **so that** users get a working context with zero extra steps.

Acceptance criteria:

- AC-1: Given a manifest declaring profile `dev` with identity seed, persona defaults, and credential asks, when the extension is installed, then `dev` exists afterward with that seed applied and the extension enabled in it — no button, no prompt beyond the normal install confirmation.
- AC-2: Given the install/update confirmation, when declared profiles will be created, then the summary names them before anything happens.
- AC-3: Given declared credential asks are unfilled, when the profile is created, then it carries a visible needs-setup signal, and first use fails closed with a plain message naming what's missing.
- AC-4: Given an update that adds a newly declared profile, when the update applies, then only the new one is created (existing ones untouched).
- AC-5: Given the install completes, when declared profiles were created, then the operator's active profile is unchanged — installation never switches context.

Edge cases:

- EC-1: Profile with the declared name already exists → bind only: placed resources apply; no seeding, no mutation of the operator's profile.
- EC-2: Operator archived or deleted a previously created declared profile → never resurrected by boot, enable/disable cycles, update, or repair of the installed extension (durable creation marker; ADR-002). A full uninstall followed by a fresh install is a new installation: it may create its declared profiles again, and an existing name is still bound, never seeded.
- EC-3: Update changes the declared seed (color, defaults) of an already-created profile → no effect on the existing profile (seed applies only at creation).
- EC-4: Uninstall → the profile, its work, config, and credentials remain; extension-owned resources withdraw.
- EC-5: Extension declares an invalid or reserved profile name → install fails validation with the exact reason (nothing half-installs).
- EC-6: Two extensions declare the same profile name → first install creates it; the second binds to it (EC-1 semantics); both markers record independently.

### US-024: Enable or disable an extension per profile

**As an** operator, **I want** extension enablement to be a per-profile switch, **so that** each context runs only the extensions it needs.

Acceptance criteria:

- AC-1: Given an installed extension, when a new profile is created, then the extension is enabled there by default.
- AC-2: Given I disable it in `finance`, when `finance` is active, then its resources, tools, and surfaces are absent; other profiles are unaffected.
- AC-3: Given enablement changes, when they apply, then they take effect without reinstall and are inspectable per profile.
- AC-4: Given an extension with secret bindings, when work resolves them, then the user binding applies by default and a profile may override per binding, with resolution scoped to the owning profile.

Edge cases:

- EC-1: Disabling in the profile the extension itself declared/created → allowed; the profile stays, resources withdraw there (operator wins).
- EC-2: Dev-linked extensions keep their workspace ties; per-profile enablement composes with them.
- EC-3: Extension disabled in every profile → effectively inert but installed; machine-level uninstall remains the removal path.

### US-025: Enable notification presets per profile

**As an** operator, **I want** one shared notification preset library with per-profile enablement, **so that** each context notifies the way that context needs.

Acceptance criteria:

- AC-1: Given presets exist, when I enable/disable them per profile, then delivery follows the active profile's set; the library itself stays shared (one per install).
- AC-2: Given a new profile, when created, then it starts from the default enablement state (enabled), adjustable immediately.
- AC-3: Given terminal or programmatic surfaces, when I list or change a preset's enablement for a profile, then structured output and named errors apply — full parity with the settings page.

Edge cases:

- EC-1: Delivery progress for profile-owned streams is profile-tagged work data — an archived profile's deliveries pause with its automations.

## Per-profile state

### US-026: Keep separate desktops and window layouts per profile

**As an** operator, **I want** my window arrangement to belong to the profile, **so that** switching contexts restores that context's desk.

Acceptance criteria:

- AC-1: Given different window layouts arranged under `dev` and `marketing` in the same workspace, when I switch profiles, then each profile's desktops and windows return as left.
- AC-2: Given a brand-new profile, when I enter it, then it starts with a clean default layout.

Edge cases:

- EC-1: Archived profile → its layouts are retained and return on unarchive.
- EC-2: Two profiles arrange the same app's windows in the same workspace → switching restores each arrangement intact; neither ever shows the other's.

### US-027: Recall profile-scoped memory

**As an** agent, **I want** machine-tier memory to be per-profile while repo-committed workspace memory stays shared, **so that** what I learn in one context doesn't contaminate another.

Acceptance criteria:

- AC-1: Given memory written at the machine tier by `marketing` work, when `dev` work recalls, then that memory is not visible; `marketing` work recalls it normally.
- AC-2: Given repo-committed workspace-tier memory, when any profile works in that workspace, then it is readable (git physics, like resource layer 4).
- AC-3: Given agent-tier memory, when the owning agent lives in a profile layer, then its memory follows that layer's visibility.

Edge cases:

- EC-1: Memory written before Profiles existed → belongs to `default`'s tier view.
- EC-2: Consolidation/maintenance respects profile boundaries — summaries never merge two profiles' machine-tier memory.

## Agents & automation

### US-028: Run under an immutable profile binding

**As an** agent, **I want** my session's profile fixed at creation, **so that** scope is stable for my whole run.

Acceptance criteria:

- AC-1: Given a session created under `dev`, when I query my context, then the profile is discoverable; everything I create belongs to `dev` (US-008.AC-2).
- AC-2: Given any structured command I run mid-session, when it touches profile-scoped reads, then my session's profile applies — there is no verb to switch my own profile.
- AC-3: Given automations, when their jobs and runs execute, then they carry the owning profile end to end (triggers, suggestions, runs).

Edge cases:

- EC-1: An agent passing an explicit profile selection for *acting* commands inside its session → the session binding wins for ownership; deterministic error if it attempts to act as another profile.
- EC-2: Reading across profiles (aggregate option) from an agent → allowed where listing surfaces allow it; labeled rows, read-only (organization, not access control).

### US-029: Converse across profiles on the shared network

**As an** agent, **I want** to talk to agents running in other profiles, **so that** contexts stay organized without silos.

Acceptance criteria:

- AC-1: Given agents in `dev` and `marketing`, when one messages the other over the network surfaces, then delivery works; the conversation is work owned by its creating side's profile.
- AC-2: Given network peer/infra registries, when profiles change, then peers are unaffected (machine-level, shared).

Edge cases:

- EC-1: A conversation's visibility in listings follows its owning profile like all work; the counterpart still sees the exchange from its own side.
- EC-2: An agent requests profile-restricted delivery or visibility → deterministic "not supported" (per-profile network isolation is future scope); no surface implies it exists.

## Foundation fixes

### US-030: Treat Global as a view, not a workspace

**As an** operator, **I want** the Global toggle to mean "across all my workspaces" and user resources to mean "available everywhere", **so that** the two stop masquerading as a fake workspace.

Acceptance criteria:

- AC-1: Given user-level resources, when I browse any workspace, then they are present (no longer visible only under Global).
- AC-2: Given the Global toggle on, when I view work, then I see items across all workspaces (an aggregate view), and creating there produces work with no workspace, owned by the active profile.
- AC-3: Given workspace registration, when I attempt to register the home directory, then it is rejected with a plain reason; the old auto-registration is gone.

Edge cases:

- EC-1: Existing installs that had the home directory as a workspace → it disappears from workspace lists; work formerly tied to it appears as no-workspace work (alpha hard cut, listed as a delete target).
- EC-2: Global toggle × profile axes compose (US-011.AC-2).
- EC-3: Zero registered workspaces → Global view is the only lens and behaves as today's fresh-start.

### US-031: Receive only in-scope session updates

**As an** operator, **I want** live session-catalog updates filtered by the daemon to my scope, **so that** nothing about other scopes ever reaches my client.

Acceptance criteria:

- AC-1: Given a live catalog subscription scoped to a workspace, when sessions elsewhere change, then no event for them arrives at my client (verified at the wire, not the render).
- AC-2: Given the profiles feature lands on top, when a profile is active, then the same server-side enforcement point filters by profile too (one filter, two axes).
- AC-3: Given an indeterminate scope, when the subscription starts, then it delivers nothing until scope is established (fail-closed).

Edge cases:

- EC-1: Reconnect/replay after a gap → replayed events obey the same scope filter as live ones.
- EC-2: Aggregate subscriptions (US-011/US-015) are explicit and labeled; they are the only way to widen a stream.

## Manageability & remote

### US-032: Manage profiles only from the machine

**As an** operator, **I want** profile management to exist only on my local machine while remote surfaces inherit scoping, **so that** pairing a device never widens who can reshape my contexts.

Acceptance criteria:

- AC-1: Given a paired remote surface, when it reads work, then the same daemon-enforced scoped/aggregate rules apply — fail-closed, owner labels in aggregate; nothing is widened for remote.
- AC-2: Given any remote surface, when any profile-state write is attempted — create/rename/archive/delete, a selection write, or an enablement change — then it is refused deterministically with the local-only reason; local surfaces are unaffected.

Edge cases:

- EC-1: Remote client holding a reference to a profile archived meanwhile → subsequent reads fall back consistently to scoped emptiness or `default`-scoped views, and the refusal/empty state names the cause.
- EC-2: There is no remote-only third read mode — remote composes the same two modes (scoped, labeled aggregate) its surface exposes.

### US-033: Inspect profiles and the active selection

**As an** agent, **I want** to list profiles and query the active selection with its winning source, **so that** I can operate deterministically inside the right context.

Acceptance criteria:

- AC-1: Given profiles exist, when I list them through terminal or programmatic surfaces, then each row carries name, identity, state (active/archived), and which one is active for my context, in structured output.
- AC-2: Given a selection query, when it resolves, then the answer names the profile and the winning source — explicit flag, environment, remembered choice of the resolved workspace, or `default` — deterministically.
- AC-3: Given a session-bound agent, when it queries, then the answer is the session's immutable binding, with the session named as source.

Edge cases:

- EC-1: Remembered choice points at an archived profile → the answer is `default`, with the fallback reason stated.
- EC-2: Query in a never-visited workspace → `default`, with "no remembered choice" as the stated source.
- EC-3: Listing when only `default` exists → one row, marked active; no error, no ceremony.
