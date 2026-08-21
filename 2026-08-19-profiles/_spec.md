# Profiles — Specification

Binding inputs: `DECISIONS.md` (D1–D27, as amended 2026-08-20), `adrs/adr-001..015.md`, `analysis/01..11` + `analysis/summary.md`, `_user_stories.md`.

---

# Part I — Product

## Overview

**Problem.** One operator runs many unrelated working contexts — product development, marketing, finance back-office — inside one undivided runtime. Every listing mixes all of it: sessions about ad copy sit between debugging sessions, automations for invoices share the attention surfaces with release loops, resource catalogs grow into one long pile, and there is no honest answer to "what did the marketing side of my life consume this month?". There is also no way to hand a teammate — or ship to a user — a ready-made division of contexts: today the division lives in the operator's head.

**Who it is for.** The operator segmenting their own contexts (primary, v1); a team member adopting a repo-committed division of contexts on their own machine; extension authors distributing complete working contexts; agents and automations, which inherit a context and operate the feature through structured surfaces. The data model is deliberately ready for less-technical personas, but v1 makes no such claim — public copy keeps that future-framed.

**Why it is valuable.** Focus: the active profile is the only thing on screen, enforced by the daemon rather than by discipline. Truthful attribution: usage and spend follow the owning context even when contexts share credentials. Shareability: the division of contexts becomes an artifact — committed to a repo for a team, or declared by an extension and materialized at install. Foundation: the same stamp-and-filter substrate is what a future multi-person era will need.

## Goals

- Operators can create, switch, rename, archive, and delete named profiles with distinct visual identities (color + icon or emoji), from the menubar switcher, Settings, and the command line.
- Every work item — session, task, loop run, automation and its runs — belongs to exactly one profile from creation, permanently; children inherit their parent's owner.
- Every listing, counter, badge, and live view shows only the active profile's work, filtered by the daemon, fail-closed; a fresh install with only `default` behaves exactly like today's product.
- Returning to a workspace restores the profile last used there; a never-visited workspace opens in `default`.
- An "All profiles" switcher state shows everything, always owner-labeled; creating there files to `default` behind a fixed, visible destination chip and an owner-naming confirmation.
- Resources compose across four additive layers (user → profile → project base → project per-profile-name), most specific winning; the two repo layers travel with the repository and light up by profile name.
- Configuration gains profile layers on both the user and project sides, with writes targeting the active profile's layer, an explicit scope override, a denylist protecting daemon identity, and "saved but not applied" feedback when a more specific layer wins.
- A profile can override provider credentials per provider through the credentials vault; attribution still follows the owning profile regardless of the key used.
- Extensions place resources into named profiles and declare profiles that are created automatically at install — no button, created exactly once, never resurrected against the operator's will.
- Phase 0 fixes land first: user resources visible in every workspace, the home directory no longer registrable as a workspace, the Global toggle meaning "across all workspaces", and session-catalog live updates filtered server-side.

## User Stories

Canonical catalog: [_user_stories.md](_user_stories.md). Index by area:

- US-001–007 — Identity & lifecycle: create, edit identity, rename (tiered), archive/unarchive, delete-empty, `default` permanence.
- US-008–013 — Scoped work: ownership at creation, fail-closed scoped reads, switching, "All profiles", create-in-`default`, per-profile usage.
- US-014–015 — Selection & return: per-workspace remembered profile, command-line selection precedence and aggregate listing.
- US-016–019 — Resource layers: user layer, profile layer, repo base + per-name folders, team adoption via hints.
- US-020–021 — Config & credentials: layered config with denylist and overridden-write feedback; per-provider credential overrides.
- US-022–025 — Extensions: placement, auto-created declared profiles, per-profile enablement, notification presets.
- US-026–027 — Per-profile state: per-profile desktops/window layouts, profile-scoped machine-tier memory.
- US-028–029 — Agents & automation: immutable session binding, cross-profile conversation on the shared network.
- US-030–031 — Foundation fixes: Global as a view, server-side filtered session catalog.
- US-032–033 — Manageability & remote: local-only management with scoped remote reads; profile and selection inspection.

## Core Features

**1. Profile identity & lifecycle.** Named profiles with color and icon/emoji chosen in a searchable picker (icons tab + emojis tab + palette with free color, in the style of Linear's team identity picker). Create from the switcher, Settings, or the command line; creating activates the new profile. Rename handles bound folders in tiers (machine folders automatic; repo folders offered as pending project changes; extension placements become dormant with hints). Archive hides a profile that owns work while preserving everything; unarchive restores; delete is only for empty profiles. `default` is permanent — every install has it, uniformly, and no one can archive, delete, or rename it.

**2. Scoped work.** Ownership is stamped at creation on every work root and inherited by children; the daemon filters every read surface to the active profile, fail-closed. Worktrees are the single ruled exception: visible in every profile, always owner-tagged. Usage and spend surfaces gain the profile dimension with the same scoping.

**3. Aggregate view.** "All profiles" is a state of the profile switcher — not a third menubar control. Rows are always owner-labeled; the state composes with the Global toggle and the workspace picker. Creation while aggregated files to `default` (fixed chip + owner toast). The daemon exposes exactly two read modes: scoped, and explicit labeled aggregate.

**4. Selection & return.** One resolution chain, evaluated after workspace resolution: explicit flag → environment variable → the workspace's remembered profile → `default`. The remembered choice stores real profiles only, per workspace (the Global lens has its own slot), and updates on every switch. Machine-scoped commands ignore profile selection entirely.

**5. Layered resources.** Four additive layers (user → profile → project base → project per-profile-name) with most-specific-wins conflict resolution and inspectable shadowing. Repo-side layers are deliberate, committed team decisions; personal material never writes into the repo. Name-binding (repo folders, extension placements) is the sharing mechanism, with dormant-until-created hints when the named profile is absent.

**6. Layered config & credentials.** Profile config layers slot into the existing overlay precedence on both machine and project sides. Writes target the active profile's layer with an explicit scope override; machine-identity keys are denylisted on profile layers; overridden writes say so. Provider credentials default to the user's single set, with per-provider overrides stored in the credentials vault under profile scope; process-environment references are rejected for profile secrets; native provider logins stay shared across profiles in v1.

**7. Extensions & profiles.** Install stays single and shared across profiles; enablement becomes per-profile (default: enabled). Each contributed resource is machine-wide or placed into a profile name; effective visibility = enabled in the profile AND placement matches. An extension may declare profiles — identity seed, persona defaults, credential asks — created automatically at install or at the update that introduces them, exactly once, recorded by a durable marker; operator lifecycle always wins (no resurrection, no seeding of pre-existing names, no mutation on update, profiles survive uninstall). Notification presets follow the same machine-library + per-profile-enablement shape.

**8. Foundation fixes (phase 0).** Global becomes a concept: user resources visible everywhere, the home directory non-registrable (its auto-registration is a delete target), the Global toggle an aggregate view whose creations are no-workspace work owned by the active profile. Session-catalog live updates filter server-side, fail-closed — the enforcement point profile scoping then extends.

**Interaction between features.** The switcher (1) drives scoping (2) and remembered selection (4); the aggregate (3) is a read mode of (2); layers (5) and config/credentials (6) both key off the active profile from (4); extensions (7) reuse (5)'s name-binding and (6)'s credential asks; everything scoped rides the enforcement point built in (8).

## Business Rules

Invariants:

1. Every work item has exactly one owning profile, assigned at creation, never changed (ADR-004).
2. Children inherit their parent's owner; no creation path can diverge from it.
3. Workspaces are machine-global; no profile owns, hides, or duplicates a workspace.
4. Scoped views are enforced by the daemon and fail closed — indeterminate scope shows nothing.
5. `default` always exists, is structurally identical to any other profile, and cannot be archived, deleted, or renamed (cosmetic identity edits allowed).
6. Exactly two read modes exist: scoped-to-one-profile and explicit owner-labeled aggregate (ADR-005).

Validation:

7. Profile names are short lowercase identifiers — letters, digits, hyphens, starting with a letter (exact bounds in Part II) — because names double as repo folder names and extension binding keys.
8. `default`, `all`, and `global` are reserved names (ADR-008).
9. Names are unique across active and archived profiles; archived profiles keep their names reserved.

Lifecycle:

10. Operator-initiated creation activates the new profile; extension-declared creation (rule 14) never changes the active profile. Skipped identity choices are auto-assigned and editable later.
11. Rename never moves work (the owner reference is stable; the name is a label). Machine-side folders rename automatically; matching repo folders are offered per project (accepted ones become pending changes the operator commits); extension placements are never edited and go dormant with hints (US-003).
12. Archive removes the profile from every selection surface, pauses its automations (listed at confirmation), blocks while its sessions run, keeps all work visible in the aggregate, and is reversible (ADR-001).
13. Delete requires zero owned work; otherwise the flow offers archive. Deletion frees the name immediately.
14. Extension-declared profiles are created exactly once per (extension instance, profile name) at install or at the update introducing them; they are never resurrected after operator archive/delete; a pre-existing name is bound, never seeded; updates never mutate an already-created profile; uninstall leaves the profile and its work (ADR-002).

Selection:

15. Resolution precedence: explicit flag → environment variable → remembered choice of the resolved workspace (Global lens has its own slot) → `default` (ADR-003).
16. The remembered choice stores real profiles only; entries pointing at archived or deleted profiles fall back to `default`; machine-scoped commands ignore selection.
17. Creating while "All profiles" is on files to `default`, with the fixed destination chip visible before commit and an owner-naming confirmation after (ADR-005).

Resources, config, credentials:

18. Resource layers are additive; the most specific layer wins a name conflict; which layer won is inspectable (ADR-006).
19. Repo per-name folders and extension placements bind by profile name; content naming an absent profile is dormant with a visible create hint; creating or renaming to the name wakes it.
20. Config writes target the layer that owns the current context (the user file under `default`, the profile file under any other profile); an explicit scope option targets user or workspace; daemon-identity keys are rejected on profile layers with the denylist reason; a write to an overridden layer reports "saved but not applied" naming the winning layer.
21. Profile-scoped secrets exist only as credentials-vault references — process-environment references are rejected for profile scope; native provider logins remain shared across profiles in v1 and surfaces say so (ADR-009). Extension secret bindings follow the same rule: user binding by default, per-profile override allowed.

Visibility and attribution:

22. No permission or grant vocabulary exists between profiles; the aggregate is always available to the operator; remote surfaces inherit read scoping and offer no profile management (ADR-010).
23. Worktrees are visible in every profile, owner-tagged (ruled exception to scoped reads).
24. Network peers and infrastructure are machine-level; conversations are work owned by the creating side's profile; agents in different profiles can converse.
25. Bridge instances belong to the profile that created them; their visibility and management follow the owner, and unattended work they deliver is owned by the instance's profile.
26. Tool-approval grants are scoped to their owning profile — an approval granted in one profile never applies in another.
27. Loop outputs are reachable only through a run the current scope can see; no surface lists outputs outside a run.
28. Desktops and window layouts are per-profile working state; a new profile starts clean.
29. The home-level agent memory tier (shipped name: global) becomes per-profile; repo-committed workspace memory is shared across profiles; agent-tier memory follows its owning agent's layer.
30. Usage and spend attribute to the owning profile regardless of which credential executed the work.

Defaults:

31. A new profile starts with: extensions enabled, notification presets at their default enablement, a clean layout, the user credentials, and no config overrides — except extension-declared profiles, which start from their declared seed (rule 14; ADR-002).
32. Work and usage recorded before Profiles existed belong to `default`.
33. There is no cap on profile count; the switcher is designed for a handful (market norm: three to six).

## User Experience

**Personas.** The operator (owns the machine, switches contexts many times a day); the team member (clones a repo carrying a profile division, manages only their own machine); the extension author (declares placements and profiles in a manifest); the agent (inherits a profile, discovers it, operates through structured commands).

**Primary flows.**

1. *First contact:* nothing. A fresh install shows a quiet, neutral switcher; everything belongs to `default`; no onboarding step is added.
2. *Second profile:* creating "marketing" (name + color + symbol, one small form) activates it; the switcher becomes a visible menubar element from that moment on — on the right side, immediately before Settings (the conventional profile position); the left cluster (mark → globe → workspace) is untouched.
3. *Daily switching:* choosing a profile refilters every surface instantly; entering a workspace restores the profile last used there; the switcher answers "work separates; project folders and machine tools are shared" in one sentence.
4. *Audit:* "All profiles" shows everything owner-labeled, composing with the Global toggle and workspace picker; creating there visibly files to `default`.
5. *Rename:* one dialog that also lists matching repo folders to rename (pre-checked) and reports extension placements that will sleep.
6. *Cleanup:* archive from Settings (or via a blocked delete) with plain copy about what stays visible where; unarchive from the archived list.
7. *Extension install:* the confirmation names the profiles the extension will create; after install the new profile is present, set up, and flagged needs-setup if credential asks are unfilled.
8. *Team adoption:* registering a cloned repo surfaces "this project declares content for profiles dev, marketing — create?"; one action per name, and the division lights up.

**Accessibility.** Color is never the only signal — name and symbol always accompany it; identity colors meet contrast floors on chips and labels; the switcher and pickers are fully keyboard-operable; the destination chip is text, not color alone.

**Onboarding and discoverability.** No new onboarding step (machine-once ruling). Discovery happens at the switcher (quiet until a second profile exists), the Settings profiles page, and the create hints (repo divisions, dormant content). The screen-by-screen map lives in `_uiux.md` (Stage 2).

## High-Level Technical Constraints

- **One runtime, one state model.** No per-profile daemons, stores, or sockets; the profile is a dimension inside the one runtime (ADR-004).
- **Server-side, fail-closed enforcement**, built on the phase-0 enforcement point; client-side filtering is never the mechanism.
- **Instant switching.** A profile switch is a filter change — no reloads, no reconnect storms, no visible latency at typical volumes.
- **Local-only management.** Remote/paired surfaces inherit read scoping; they cannot create, rename, archive, or delete profiles in v1.
- **Agent/operator manageability outcome.** Everything the UI can do to profiles is equally doable outside it: full lifecycle, selection resolution (including "which profile am I in and why"), scoped and aggregate listings, effective-layer inspection for resources/config/credentials — all with structured output and deterministic, named errors. Machine-scoped commands ignore profile context by contract.
- **Extension ecosystem expectation.** Placement, declared profiles, per-profile enablement, and credential asks are manifest-level, validated at build/install, and honored by every install source; nothing is copied into profiles.
- **Configuration lifecycle.** The two new profile config layers enter the documented precedence with validation, examples, and feedback semantics ("saved but not applied"); the denylist is part of the contract.
- **Privacy & security.** Profile credential overrides resolve only within their owning profile's scope; profile secrets never ride process-wide environment; copy states plainly that profiles are separation, not security.
- **Copy governance.** Bare "Profile" names only this concept (ADR-008); existing compound names stay; non-technical-audience claims stay future-framed; maturity labels follow the copy standards.
- **Vocabulary mapping.** What shipped documentation calls the global tier of resources and memory is the **user** layer in this document (config scope value `user`, mirroring the Claude Code convention); after phase 0, "Global" names only the cross-workspace view. "Machine" remains only for daemon identity (denylisted keys, machine-scoped commands). Whether stored scope names change — and the delete targets if so — is resolved in Part II.

## Non-Goals (Out of Scope)

- **Serving non-technical users as a claim.** Profiles build the foundation; install path and first-run remain technical. Copy stays future-framed (governance rule).
- **Access control between profiles.** No grants, locks, or hiding; single-operator machine; the frozen cross-workspace-access vocabulary stays closed (ADR-010).
- **Moving work between profiles.** The stamp is immutable; an administrative transfer verb is recorded future scope (archive covers cleanup; ADR-001).
- **Per-profile network isolation.** Cross-profile agent conversation is desired; peers/infra stay machine-level (ADR-010).
- **Inbound routing table.** No provenance-based routing (channel/thread/sender → profile) in v1: unattended inbound work is owned by its entry point's profile — bridge deliveries by the bridge instance's owner, automation runs by their stamped owner. One entry point serves one profile; the route table is recorded future scope (ADR-011).
- **Per-profile provider logins/homes.** Native provider CLI logins remain single, shared across profiles; per-profile provider homes are a recorded future rung (ADR-009).
- **Per-profile scheduler fairness or budgets.** Concurrency budgets stay machine-level in v1.
- **Hiding worktrees (or anything) per profile.** Capability gating is future scope; v1 tags and shows.
- **Per-profile onboarding.** Onboarding stays machine-once; the groundwork for a per-profile path is a data-shape concern only.
- **Remote profile management.** Local-only in v1; remote inherits read scoping.
- **Renaming existing "profile" compounds.** Task profile, layout-profile, sandbox profile, Trust Profile, connection profile keep their names (ADR-008).
- **Read-only aggregate / destination picker.** Both considered and rejected in favor of create-in-`default` with a fixed chip (ADR-005).
- **Multi-person use of one daemon.** A future era; this design's stamps and fail-closed filtering are its deliberate substrate, nothing more.

## Open Questions

None. Every product-level branch raised by the exploration (`analysis/summary.md` gating and second-tier lists) was resolved in the grill-me decision log (`DECISIONS.md` D1–D27, as amended) or in the spec Stage 1 rounds (ADR-001..003, ADR-005 chip form, ADR-007 scope, ADR-011 inbound routing). Surface and implementation questions belong to Stage 2.

---

# Part II — Technical

Designed to serve the frozen surfaces: [`_dx.md`](_dx.md) and [`_uiux.md`](_uiux.md). Authoritative scope contract: `analysis/10_analysis_profile-scope-inventory.md` (17 stamped roots; A/A-inh/B/C/D classes); verified anchors: `analysis/11_arch-path-map.md`.

## Executive Summary

Profiles become a first-class daemon domain (`internal/profile`) with a new catalog and its supporting state tables (selections, lifecycle journal + seed snapshots, delivery permits, enablement/marker/mute rows), a `profile_id` stamp on exactly 17 creation-root tables (61 descendants inherit through existing composite FKs — no new columns), and one server-side read-scope contract with exactly two modes (scoped, labeled aggregate) enforced at the store layer, fail-closed. Resources, config, and credentials gain profile layers by extending three existing seams — discovery roots, the config overlay chain, and the vault ref grammar — never by new parallel mechanisms. Extensions gain per-profile enablement (join table), per-resource placement, and install-time declared-profile creation with durable create-once markers. The shipped `global` scope word is hard-cut to `user` across stored values and payloads in the same change (no aliases). Phase 0 deletes the boot-time registration of `~/` as a workspace and moves session-catalog filtering server-side — the enforcement point every profile read reuses.

## MVP Boundary

MVP = build phases 0–7 below, complete: foundation fixes, profile domain + stamping, selection/resolution, enforcement sweep, resource/config/credential layers, extension integration (placement + declared profiles + per-profile enablement), web surfaces, docs/skill/codegen. Post-MVP is exactly the recorded future scope in Part I Non-Goals (transfer verb, route table, per-profile provider homes, capability gating, per-profile scheduler fairness, remote management). Out of scope: everything Non-Goals names. Task numbering lands in `_tasks.md` (`cy-create-tasks`); the final two tasks are the QA pair, with browser e2e (this spec is UI-bearing).

## Developer Experience

- [Developer experience contract](_dx.md) — golden path; `compozy profile` verb set + `--all-profiles`; selection precedence; resource folders; per-profile credentials; extension enablement; HTTP/UDS routes (lifecycle, selection, enablement, scoped reads); extension manifest; config layers; native tools; deterministic errors.
- [UI change map](_uiux.md) — 13 surfaces (S1 switcher … S13 preset enablement), component plan (`SymbolPicker` + `web/src/systems/profiles/` composites), 7 artboards. UI-bearing signal for the QA tail.

## System Architecture

- **`internal/profile` (new)** — profile catalog (CRUD, identity, lifecycle state machine), selection store, resolution result types, name grammar, reserved names. Owns the `profiles` + `profile_selections` tables. Analogous in rank to `internal/workspace`: a leaf domain other packages reference by ID only.
- **Resolution** — CLI: profile resolution joins the existing workspace-resolution boundary (`internal/cli/workspace_resolution.go` idioms: source taxonomy, typed error payload, context-key record, JSONL resolution frame). API: explicit query params validated at `internal/api/core` handlers; agents: derived from session identity, never accepted from the caller.
- **Enforcement** — store-layer read scope: every A-class list/get/stream query takes an explicit `(profileID string, allProfiles bool)`; the session-catalog stream gains a subscriber filter (the D4 fix) that phase 3 extends with the profile axis. No middleware magic, no context-injected defaults below the API boundary.
- **Layers** — resources: two new discovery roots in `WorkspaceDiscoveryRoots` + widened `resource_records` scope; config: two new overlay layers in `loadWithHome` + two new `WriteScope` values + profile-overlay validator (denylist); credentials: `profiles` vault namespace + `PreStartScope.ProfileID`.
- **Extensions** — enablement join table read at kit-publish time; placement filter at surface projection; declared profiles executed by the install/update pipeline with markers.
- **Web** — server-backed selection (API) cached in a client store; profile-qualified query keys; stream generation bump on switch; new `systems/profiles` composites; desktops partition by profile inside the existing clientstate aggregate key. Two distinct states, never conflated: the daemon's **remembered choice** (ADR-014) vs each client's **ephemeral active view** (profile or aggregate, per lens) — `profile.selection_changed` invalidates the remembered projection and never force-switches an open client (US-010.EC-4).
- **Data flow (scoped read)**: client/CLI → handler validates profile params (default = `default`) → store method with explicit scope → rows; aggregate mode adds owner join (profile name/color/icon) for labels.

## Architectural Boundaries

- New package `internal/profile`. May import: `internal/store/globaldb`, `internal/config` (paths), `internal/logger`. Must NOT import: `daemon`, `api/*`, `cli`, `session`, `task`, `workspace`, `extension` (no back-pointers; consumers pass IDs).
- `internal/session`, `internal/task`, `internal/loop`, `internal/automation`, `internal/bridges`, `internal/worktree`, `internal/network`, `internal/notifications`, `internal/observe`, `internal/tools` accept `profileID` as data (params/fields) — none imports `internal/profile`.
- `internal/cli`, `internal/api/core|httpapi|udsapi`, `internal/daemon` import `internal/profile`; `internal/daemon` remains the only composition root (boot: ensure default profile, ensure layout, wire service into handlers/deps).
- `internal/config` gains profile path derivation + overlay options without importing `internal/profile` (takes the profile name/ID as a string option).
- `magefiles/boundaries.go` gains the `internal/profile` rules in the same commit that creates the package.

## Implementation Design

### Core Interfaces

`internal/profile` exposes a **concrete** `*profile.Manager`; interfaces are defined where consumed (house rule): `api/core` declares its narrow `profileLifecycle`/`profilePlans` seams, `cli` deps declare `profileResolver`, the extension pipeline declares `declaredProfileCreator` — each with a compile-time assertion against `*profile.Manager` at daemon wiring.

```go
// internal/profile — concrete domain type (consumers define their own interfaces)
type Manager struct{ /* store, home paths, vault rewriter, per-profile op mutex */ }

// Methods (signatures final):
// Create(ctx context.Context, in CreateInput) (Profile, error)                     — operator; activates via Activate
// CreateDeclared(ctx context.Context, in DeclaredInput) (Profile, error)           — extension; never activates (ADR-002)
// UpdateIdentity(ctx context.Context, name string, p IdentityPatch) (Profile, error)
// PrepareRename(ctx context.Context, name, newName string) (RenamePlan, error)
// Rename(ctx context.Context, name string, opts RenameOptions) (RenameResult, error)   // revision inside opts
// PrepareArchive(ctx context.Context, name string) (ArchivePlan, error)
// Archive(ctx context.Context, name, planRevision string) (ArchiveResult, error)
// Unarchive(ctx context.Context, name string) (UnarchiveResult, error)
// PrepareDelete(ctx context.Context, name string) (DeletePlan, error)
// Delete(ctx context.Context, name, planRevision string) (DeleteResult, error)
// Resolve(ctx context.Context, in ResolveInput) (Resolution, error)   — store-backed; rules below
// List(ctx context.Context) ([]ProfileWithCounts, error)              — counts only; `current` is surface decoration
// GetByName(ctx context.Context, name string) (Profile, error)
// ListOps(ctx context.Context) ([]LifecycleOp, error)                 — pending + failed + recent ops (ops surface)
// RetryOp(ctx context.Context, opID string) (LifecycleOp, error)      — the only transition out of `failed`

type Profile struct {
    ID         string // ULID, immutable; the stamp value
    Name       string // unique handle: ^[a-z][a-z0-9-]{0,31}$; reserved: default, all, global
    Color      string // #rrggbb
    Icon       string // exactly one of Icon/Emoji is set
    Emoji      string
    State      State  // active | archived
    CreatedAt  time.Time
    ArchivedAt *time.Time
}
```

```go
// Inputs — every field final; validation noted inline
type CreateInput struct {
    Name, Color, Icon, Emoji string // symbol auto-assigned when both empty; icon/emoji exclusive
    Activate                 *Lens  // operator path: lens whose remembered choice activates; nil = none
}
type DeclaredInput struct { // extension declared profile (ADR-002): marker-gated, no activation
    Extension string        // installed instance name (marker key component)
    Name      string
    Seed      DeclaredSeed  // identity + persona defaults + credential asks; applied at creation only
}
type IdentityPatch struct{ Color, Icon, Emoji *string } // nil = unchanged (color); setting Icon or Emoji REPLACES the symbol — the counterpart clears in the same update (exclusivity preserved atomically)
type RenameOptions struct {
    NewName      string
    Repos        RepoChoice // all | none | selected workspace ids
    PlanRevision string     // from PrepareRename; stale ⇒ profile_plan_stale
}
```

```go
// Plans back the frozen dialogs and CLI flows (read-only, revisioned)
type RenamePlan struct {
    Revision          string
    MachineFolders    []string        // renamed automatically in finalize
    RepoCandidates    []RepoFolderRef // offered per workspace; operator commits
    DormantPlacements []PlacementRef  // extension placements that will sleep
    VaultRefRewrites  int
}
type ArchivePlan struct{ Revision string; RunningSessions []string; LeasedRuns, QueuedRunsToFreeze int; AutomationsToPause []string }
type DeletePlan struct{ Revision string; Removed RemovalSummary; SelectionsToSweep int }
type RemovalSummary struct{ Agents, Skills, Loops, MCPServers, ConfigKeys, CredentialOverrides, MemoryEntries, DesktopPartitions int } // the total removal catalog — every profile-owned personal resource and config sidecar; preview == applied, byte-for-byte in meaning
type RenameResult struct{ RepoResults []RepoRenameOutcome; DormantPlacements []PlacementRef }
type ArchiveResult struct{ PausedAutomations []string; FrozenQueuedRuns int }
type UnarchiveResult struct{ Profile Profile; PausedAutomations []string } // paused list repeated for explicit reactivation
type DeleteResult struct{ Removed RemovalSummary; SweptSelections int }
```

```go
// Support types (final)
type Lens struct{ Kind SelectionLens; WorkspaceID string } // "workspace" requires WorkspaceID; "global" forbids it
type DeclaredSeed struct {
    Color, Icon, Emoji string          // same exclusivity rule as CreateInput
    Defaults           PersonaDefaults // defaults.agent/provider/sandbox (D15)
    CredentialAsks     []CredentialAsk // surfaced as needs-setup until filled
}
type PersonaDefaults struct{ Agent, Provider, Sandbox string }
type CredentialAsk struct{ Provider, Slot string }
type RepoChoice struct{ All, None bool; WorkspaceIDs []string } // exactly one form set; validated
type ProfileWithCounts struct{ Profile; WorkItems int; NeedsSetup bool } // work_items contract; NeedsSetup from profile_credential_requirements; `current` is decorated at the surface from the caller's own Resolution
type CredentialRequirement struct{ Provider, Slot, SourceExtension string; Missing bool } // GetByName projection detail
type RepoFolderRef struct{ WorkspaceID, WorkspaceName, Path string }
type PlacementRef struct{ Extension, Resource, ProfileName string }
type RepoRenameOutcome struct{ WorkspaceID string; Renamed bool; Reason string }
type ResolveInput struct {
    Flag, Env        string // --profile / COMPOZY_PROFILE values
    SessionProfileID string // daemon-validated session identity, when present
    Lens             Lens   // the resolved workspace or global lens
}
type LifecycleOp struct{ ID, Kind, Profile, Status, Step, Error string } // ops surface row (op_-prefixed id)
```

```go
// internal/profile/selection.go — the remembered choice (ADR-003, ADR-014)
type SelectionLens string // "workspace" | "global"

type Selection struct {
    Lens        SelectionLens
    WorkspaceID string // "" for the global lens
    ProfileID   string
}

type SelectionStore interface {
    Get(ctx context.Context, lens SelectionLens, workspaceID string) (Selection, bool, error)
    List(ctx context.Context) ([]Selection, error) // the full workspace→profile map (Settings, API)
    Put(ctx context.Context, sel Selection) error
    SweepProfile(ctx context.Context, profileID string) error // delete cleanup only (archive preserves fallback provenance)
}
```

```go
// internal/profile/resolve.go — one resolver, evaluated after workspace resolution
type ResolutionSource string // "flag" | "env" | "remembered" | "session" | "default"

type ResolutionNote string // "" | "archived_remembered_fallback" | "no_remembered_choice"

type Resolution struct {
    Profile Profile
    Source  ResolutionSource
    Note    ResolutionNote // why a fallback happened; surfaces render it (US-033.EC-1/2)
}

// Resolve is a Manager method (store-backed). Rules, in order:
//   1. A daemon-validated SessionProfileID is authoritative (source "session"); a differing
//      Flag/Env on an acting path fails typed profile_session_conflict — never silently ignored.
//   2. Otherwise: flag → env → remembered(lens) → default.
// Fail-closed: unknown/archived/unavailable explicit selection is a typed error, never a fallback.
```

```go
// Read-scope contract threaded to every A-class store read (no context defaults)
type ReadScope struct {
    ProfileID   string // exactly one profile when AllProfiles is false
    AllProfiles bool   // explicit labeled aggregate (the only widening)
}
// func (s ReadScope) Validate() error — rejects BOTH invalid states before any query runs:
// empty ProfileID with AllProfiles=false, and ProfileID set with AllProfiles=true.
// internal/tools/tool.go — Scope gains one field reaching every native tool projection
// Scope{ WorkspaceID, SessionID, AgentName, ActorKind, Operator, ProfileID string }
```

### Data Models

Side-table-vs-JSON: every new datum below is matchable state → columns and side-tables; zero JSON metadata blobs.

New definition file `internal/store/globaldb/schema/definitions/05_profiles.sql`. Two consecutive append-only globaldb migrations keep the ADR-007 phase gate real — **`00078_schema.sql` (phase 0, foundation)** and **`00079_schema.sql` (profiles)** — and the memory stream ships its own migration (ADR-013). Contents:

- `profiles` (in `00079`) — `id TEXT PK` (ULID; stable stamp survives rename) · `name TEXT NOT NULL UNIQUE` (public handle + binding key) · `color TEXT NOT NULL` · `icon TEXT` / `emoji TEXT` (`CHECK ((icon IS NULL) <> (emoji IS NULL))` — exactly one symbol, auto-assigned before insert when skipped) · `state TEXT NOT NULL CHECK (state IN ('active','archived')) DEFAULT 'active'` · `created_at` · `archived_at`. **`00079` itself seeds the `default` row** — fixed constant id, the zero ULID `00000000000000000000000000` (ADR-012; the `scheduler_pause` singleton precedent), with its persisted identity (`color #8E8EB5`, `icon circle`) — so the stamp backfill later in the same migration has its FK target on both fresh and upgrade paths; boot's ensure step only verifies. App layer enforces grammar + reserved names (`default`, `all`, `global`).
- `profile_selections` — `lens TEXT CHECK (lens IN ('workspace','global'))` · `workspace_id TEXT NOT NULL DEFAULT ''` · `profile_id TEXT NOT NULL REFERENCES profiles(id)` · `updated_at` · `PK (lens, workspace_id)`; `AFTER DELETE ON workspaces` cleanup trigger (house template `33_extensions.sql:22-43`). Stores real profiles only (aggregate is never persisted).
- **Stamp on the 17 roots** (scope contract, analysis/10; in `00079`): `sessions`, `tasks`, `loop_runs`, `automation_jobs`, `automation_triggers`, `automation_suggestions`, `bridge_instances`, `worktrees`, `network_channels`, `network_direct_rooms`, `network_threads`, `network_work`, `notification_cursors`, `tool_approval_grants`, `event_summaries`, `dead_entities`, `token_usage_daily` — each gains `profile_id TEXT NOT NULL REFERENCES profiles(id)`; `00079` adds nullable → backfills to the `default` profile id → rebuilds NOT NULL. Immutability: `BEFORE UPDATE OF profile_id … RAISE(ABORT)` trigger per root (D9). **Active-owner creation guard**: each root also gains a `BEFORE INSERT` trigger — `RAISE(ABORT)` unless `profiles.state = 'active'` for `NEW.profile_id` **and** no pending `profile_lifecycle_ops` row exists for it (availability gate) — the shared transactional guard at every creation boundary (manual, automation, bridge delivery, spawn all converge on it; store layers map the aborts to the typed `profile_archived` / `profile_unavailable` errors). Children: **no column** — inheritance through existing FKs (61 tables untouched).
- Identity-bearing PK changes (greenfield hard cut, backfill `default`; in `00079`): `token_usage_daily` PK → `(day, profile_id, workspace_id, agent_name)` + index `(profile_id, day)`; `notification_cursors` PK → `(scope_kind, profile_id, workspace_id, consumer_id, stream_name, subject_id)`; `bridge_instances` stamp lands in **both** `InsertBridgeInstance` and `UpsertBridgeInstance` (identical column lists — path map §10).
- `extension_profile_enablement` — `extension_name TEXT` · `profile_id TEXT REFERENCES profiles(id)` · `enabled INTEGER NOT NULL` · `PK (extension_name, profile_id)`; row absent = enabled (default-on, D17); cleanup triggers on extension delete + profile delete. **The old global authorities die in `00079`** (hard cut): `extensions.enabled` and `notification_presets.enabled` columns are dropped via table rebuild, their indexes/queries/DTO fields/CLI+API behaviors are delete targets, and any pre-profile disabled state migrates into explicit `default`-profile exception rows — the per-profile tables become the only enablement authority across runtime, HTTP, UDS, CLI, Web, docs, and generated types.
- `extension_profile_markers` — `extension_name TEXT` · `profile_name TEXT` · `created_profile_id TEXT` · `created_at` · `PK (extension_name, profile_name)`; create-once record (ADR-002); removed with the extension instance on uninstall; **survives profile archive/delete** (that is its purpose).
- `notification_preset_enablement` — `preset_name` · `profile_id` · `enabled` · `PK (preset_name, profile_id)`; **absent row = enabled** (uniform default-on for every preset in every profile — the frozen US-025.AC-2/S13 behavior, same shape as extension enablement); rows exist only for disables.
- `profile_lifecycle_ops` — `id TEXT PK` (opaque `op_`-prefixed ULID; stored and public form identical) · `kind CHECK (kind IN ('create','rename','archive','unarchive','delete'))` · `profile_id` · `old_name` · `new_name` · `plan_revision` · `status CHECK (status IN ('applied','finalizing','done','failed'))` · timestamps; plus `profile_lifecycle_op_steps` — `op_id` · `seq` · `action` · `path_old` · `path_new` · `status` · `PK (op_id, seq)`. The durable journal written inside the apply transaction (Lifecycle Operation Protocol); typed columns, no JSON. **Non-`done` rows reserve their old/new names and derived filesystem paths** against concurrent create/rename until finalize completes. Declared creations additionally persist their **seed snapshot** atomically with the op — `profile_lifecycle_op_seed` (`op_id PK`, color/icon/emoji, `default_agent`, `default_provider`, `default_sandbox`, `declaration_digest`) + `profile_lifecycle_op_credential_asks` (`op_id, provider, slot`) — so finalize and retry reproduce the exact accepted declaration without ever re-reading mutable extension state (ADR-002's create-once survives any crash boundary).
- `profile_credential_requirements` (in `00079`) — the **durable live authority behind needs-setup** after the op reaches `done`: `profile_id REFERENCES profiles(id)` · `provider` · `slot` · `source_extension` · `declaration_digest` · `created_at` · `PK (profile_id, provider, slot)`; populated atomically from the accepted seed snapshot in the apply transaction. A requirement is *missing* while no matching `vault:profiles/<name>/providers/<provider>/<slot>` secret exists (the canonical vault mutation path is what satisfies it — no second write path); `needs_setup` = any missing requirement, exposed through the shared list/get projections on HTTP/UDS/CLI/Web. Extension updates never touch existing rows (seed applies at creation only); uninstall leaves them (they describe the profile, not the package); profile delete removes them (cleanup trigger).
- `resource_records.scope_kind` widens to `('user','workspace','profile','workspace_profile')` — `user` renames shipped `global` (ADR-013), `profile` carries `scope_id = profiles.id`, `workspace_profile` carries `scope_id = '<workspace_id>@pf:<profile_name>'` (name-bound layer 4; the `@pf:` segment reuses the validated extension-data segment grammar). Grant ceilings form a **lattice, not a total order** (D24 re-derivation): `user` is the top; `workspace` and `profile` are incomparable branches; `workspace_profile` is their meet (intersection). `scopesThrough` is the downward closure — `user → {user, workspace, profile, workspace_profile}` · `workspace → {workspace, workspace_profile}` · `profile → {profile, workspace_profile}` · `workspace_profile → {workspace_profile}` — and narrowing two ceilings takes their **meet** (`meet(workspace, profile) = workspace_profile`); scalar rank minimums are forbidden — they widen one axis. Closure and pairwise-meet tables are pinned in the canonical extension capability suite, same change as the enum.
- Memory stream (own SQL migration, ADR-013): `memory_catalog_entries.scope` (and peers + indexes) hard-cut `'global'` → `'profile'`, **and profile ownership becomes durable identity**: `profile_id TEXT NOT NULL DEFAULT ''` joins the catalog, consolidations, decisions, events, recall-signal tables and every unique/FTS index that can expose or combine profile-tier rows (catalog identity → `(workspace_id, profile_id, scope, agent_name, agent_tier, type, slug)`). Former-global rows backfill to the default profile's zero-ULID; workspace-tier rows carry `''` (shared by design, D14); agent-tier rows carry the owning agent's layer (`''` for user/workspace-layer agents, the profile id for profile-layer agents). Recall/FTS/cache identities include it — the persisted predicate behind the memory matrix rows. The directory move is **durably coupled to the stream version**: the migration writes a pending row into `memory_maintenance_ops(op PK, status, created_at, completed_at)` (`op = 'move_global_dir'`); the memory store's open path executes the atomic same-volume rename `$COMPOZY_HOME/memory` → `profiles/default/memory/` (idempotency guard: target exists ∧ source absent ⇒ done), marks the row done, and **refuses profile-tier reads while the row is pending** (fail-closed). Row-driven, deterministic, convergent on re-run — reconcile-shaped, not boot repair. The old path is a delete target, never a fallback read. Workspace/agent tiers unchanged (D14).
- Stored scope-bearing rows and refs rewritten in `00079` (ADR-013): `resource_records` `'global'` rows → `'user'`; `mcp_auth_tokens` / `mcp_oauth_registrations` scope CHECKs widen and their rows rewrite; stored `vault:mcp/global/…` and `vault:extensions/global/…` refs rewrite to `…/user/…` (ref-prefix UPDATE inside the migration, equivalence-tested).
- `notification_delivery_permits` (in `00079`) — the durable owner-active permit (Invariant 18): PK = the full cursor identity + `delivery_id` (`scope_kind, profile_id, workspace_id, consumer_id, stream_name, subject_id, delivery_id`) · `acquired_at`; acquire = INSERT in an immediate transaction that verifies the owner is active; clear = DELETE inside the cursor-advance transaction; archive refuses while owner rows exist; the notifications service replays surviving rows by delivery id at boot.
- `extension_env_bindings` is **rebuilt** in `00079` with a profile dimension (hard cut of its current `(extension_name, workspace_id, env_name)` identity — delete target 13): PK `(extension_name, profile_id DEFAULT '', workspace_id DEFAULT '', env_name)`; `''` = the unscoped axis value. **Resolution order for an acting (profile, workspace) pair is total and specificity-first with the profile axis outranking the workspace axis** (D7 makes the profile override primary): `(profile, workspace)` → `(profile, '')` → `('', workspace)` → `('', '')`; first present row wins, absence falls through, no row = the manifest's declared requirement stays unbound. Existence trigger for non-`''` profile ids + cleanup triggers on extensions, workspaces, **and** profiles; id-keyed → rename never rewrites bindings.
- **Session creation identity**: the stable profile id joins `SessionCreationProfile`'s canonical digest input (the immutable creation witness — scope contract change target 6); every creation builder, reseed path, and goal-binding witness threads it; the old digest input shape is delete target 14; `loop_runs.origin_creation_profile_ref` (the *digest ref*, unrelated naming) is untouched.
- `attention_workspace_mutes` (in `00079`) — `profile_id REFERENCES profiles(id)` · `workspace_id REFERENCES workspaces(id)` · `PK (profile_id, workspace_id)` + cleanup triggers on both parents; replaces the global `attention.muted_workspaces` array (delete target 5) with typed per-profile rows.
- **No-workspace representation (phase 0, `00078`)** — one rule, four shapes, chosen so no durable identity ever loses uniqueness or FK enforcement (NULL inside composite keys/UNIQUEs would disable both):
  1. **Plain nullable workspace FKs not used in any key** keep the shipped pattern — `NULL` + a scope discriminator (`tasks`, `task_runs`, automation tables, `bridge_instances`; `70_tasks.sql:92-93`).
  2. **`sessions` (real user-facing scope semantics)**: `workspace_id` stays `NOT NULL`, `''` = no-workspace, plus a true discriminator — new `scope TEXT NOT NULL CHECK (scope IN ('global','workspace'))` paired by `CHECK ((scope='workspace') = (workspace_id <> ''))`; the `REFERENCES workspaces` is swapped for the trigger pair (`BEFORE INSERT/UPDATE` existence check for non-`''` + `AFTER DELETE ON workspaces` cascade — the `extension_env_bindings` template).
  3. **Identity-bearing columns with no scope semantics of their own** stay `NOT NULL` with **`''` alone** as the documented no-workspace value (no discriminator column): `session_prompt_admissions` + `session_health` (UNIQUEs keep deduping), `agent_soul_*`/`agent_heartbeat_*`, `tool_approval_grants`, `dead_entities`, the network families' key-bearing columns, the loop family — whose `CHECK (length(trim(workspace_id)) > 0)` guards are **rebuilt to admit `''`**, each enumerated in the migration — and `notification_cursors`/`token_usage_daily`/audit + stat rows (workspace inside PK/denormalized). Tables that carried `REFERENCES workspaces` swap it for the same trigger pair as shape 2.
  4. **Rows meaningless without their workspace** are dropped or asserted empty (table below).
  The implementing migration states each family's exact resulting DDL (columns, CHECK rebuilds, trigger pairs, unchanged UNIQUE/PK identities); the declarative schema is the single end-state authority and the equivalence suite (IT-001/IT-003) proves it. The legacy `~/` workspace row is deleted only after the disposition table has rewritten everything it owns, so its `ON DELETE CASCADE` edges fire over nothing.

#### Phase-0 home-workspace disposition (migration `00078`)

Every `REFERENCES workspaces` family is enumerated with a fate before the `~/` row is deleted. Fates: **rewrite** (rows survive as no-workspace: column rebuilt nullable where NOT NULL today, home rows set NULL), **drop-row** (meaningless without its workspace), **assert-empty** (home rows are invalid by construction; the migration aborts loudly if any exist).

| Family (definition file) | Fate |
| --- | --- |
| `sessions` (20) | rewrite — **shape 2** (`''` + `scope` column + trigger pair) |
| `session_prompt_admissions` + `session_health` (20) | rewrite — **shape 3** (`''` alone; UNIQUEs keep deduping) |
| `tasks`, `task_runs` (70/72 — already nullable) | rewrite — set NULL (`scope='global'` already models it) |
| `automation_jobs` / `automation_triggers` / `automation_runs` refs (30 — nullable) | rewrite — set NULL |
| `bridge_instances` + FK children (31 — nullable) | rewrite — set NULL |
| loop family (50 — plain `NOT NULL` columns, several inside PKs) | rewrite — **shape 3** (`''` alone; every `length(trim())` CHECK rebuilt) |
| `agent_soul_*` (11), `agent_heartbeat_*` (10 — NOT NULL) | rewrite — **shape 3** (`''` alone + trigger-pair swap; user-layer agents are workspace-less post-D3) |
| `tool_approval_grants` (42), `dead_entities` (41) | rewrite — **shape 3** (`''` alone + trigger-pair swap) |
| `event_summaries` (40) + `token_usage_daily` (20 — workspace inside PK) + network stat/count rows (60) | rewrite — **shape 3** (`''`, denormalized identities preserved) |
| `network_audit_log` + `permission_log` (60/21) | rewrite — **shape 3** (`''`; diagnostics preserved, never erased) |
| `network_channels` / `network_threads` / `network_work` / `network_direct_rooms` + key-bearing children (60) | rewrite — **shape 3** (`''` alone + trigger-pair swap; composite identities preserved) |
| `workspace_network_coordination` (60 — PK workspace_id) | drop-row |
| `worktrees` + children (15) | assert-empty (a worktree of `~/` is invalid by construction) |
| `notification_cursors` (37 — workspace_id inside PK) | rewrite — home rows re-keyed to `''` |
| `extension_env_bindings` home rows, `extension_dev_links` home rows (33/34) | drop-row (workspace-tied dev/binding state) |
| `clientstate` home partition (daemon store) | purge via `PurgeWorkspace` |

The implementing task inventories every physical `workspace_id` occurrence (FK or denormalized) against this table — a column with no row here fails the task, not the operator. Fixtures: **one positive fixture** seeding every rewrite/drop family proves counts, identities, and relationships survive `00078` (IT-003); **one negative fixture per assert-empty family** proves the migration aborts loudly (IT-075).
- Config write model: `WriteScope` = `user` (renamed from `global`) | `profile` | `workspace`; `settings.ScopeKind` = `user | profile | workspace | agent` — **the shipped `agent` scope is untouched** (agent-local skills/settings sources keep their mappings); only the obsolete `global` value dies in both enums. `resolveWriteTarget` gains two arms; the profile-overlay validator rejects the denylist sections/keys (`http.*`, `daemon.*`, `log.*`, `database`, `gateway`, `shell`, `marketplace`, `observability`, `network`, `sandboxes`) — fail-closed like `validateWorkspaceConfigOverlay`.
- MCP sidecars gain the same two layers as config: `~/.compozy/profiles/<p>/mcp.json` and `<ws>/.compozy/profiles/<p>/mcp.json`, merged in their owning config layer's slot (five-layer order preserved), written through the same context-owned write-target rule, keyed by server name, orphan files a diagnostic (never half-applied); machine-side files move with the profile folder on rename, project-side files are committed team state; MCP auth/registration scope coupling widens in `00079`.
- Vault: `profiles` namespace joins `vaultRefPattern` + `supportedNamespaces`; grammar `vault:profiles/<name>/providers/<provider>/<slot>` and `vault:profiles/<name>/extensions/<ext>/<key>` with owner-prefix containment (mcp_refs template); rename rewrites `vault:profiles/<old>/…` refs atomically in the rename transaction (daemon-owned store; ADR-012).
- `providers` pre-start cache: `PreStartScope.ProfileID` added; cache key composition includes it (no cross-profile credential reuse).
- Desktops: window-manager clientstate aggregate keyed by `(workspace_id, profile_id)` — partition of the existing per-workspace snapshot document (no globaldb schema; path-map correction 11).
- API contract: new spec files under `internal/api/spec/` for every route in `_dx.md`; `CatalogEvent` gains `ProfileID`/`ProfileName`; listing rows gain `profile_name` in aggregate mode; `make codegen` regenerates TS types.

### Lifecycle Operation Protocol

SQLite rows, selection rows, vault refs, profile directories, and config files cannot commit as one atom — lifecycle mutations therefore run **prepare → apply → finalize**, patterned on the workspace `UnregisterPreparer` two-phase flow:

1. **Prepare** (read-only): build the `Plan` — exact row mutations, directory operations, ref rewrites — plus a `Revision` fingerprint over everything read (profile row version, folder digest, matching repo folders). Plans feed the plan endpoints, the frozen dialogs, and TTY confirmations.
2. **Apply** (authoritative): one `BEGIN IMMEDIATE` transaction re-validates `Revision` (stale ⇒ `profile_plan_stale`, nothing commits) and commits every row mutation — catalog, state transition checks (running sessions, leased runs), automation pauses, selection sweeps (delete only — archive preserves rows), vault ref rewrites, enablement/marker rows — **plus the lifecycle journal**: the `profile_lifecycle_ops` row and its `profile_lifecycle_op_steps` (Data Models), carrying the op kind, old/new names, revision, and every contained filesystem action. The journal is what survives a crash, and its non-`done` rows are **durable reservations**: the apply transaction of any create/rename/delete also verifies that the requested name matches no pending op's `old_name`/`new_name` (typed `profile_name_taken` naming the operation as holder) — so recovery can never touch a replacement profile's paths. Live ops serialize on the reserved names and derived paths as well as the profile id; the in-process mutex covers live ordering only.
3. **Finalize** (contained filesystem, **forward-only**): journaled steps run in order under containment checks, each idempotent, each checked off in the journal. There are no post-commit compensations — once apply commits, recovery is forward-only. Boot reconciliation (composition root) **auto-resumes only `applied` and `finalizing` ops**; a `failed` op stays reserved and unavailable until the explicit retry transition (`RetryOp` — the same primitive behind the ops routes and CLI), so an operator-visible failure never restarts itself. A terminal step failure marks the op `failed`, emits `profile.lifecycle_op_failed`, and surfaces in diagnostics; pre-commit failures simply roll the transaction back (no filesystem was touched, no journal row exists).
4. **Repo edits never enter the protocol**: accepted repo folder renames are reported as pending working-tree changes — explicitly non-atomic, the operator commits them.
5. Operator `Create` activates through its explicit `Activate` lens input; `CreateDeclared` carries none and never touches selections.
6. **Availability gate**: a profile with any `profile_lifecycle_ops` row not in `done` is **unavailable** — the resolver, every work-creation boundary (the per-root triggers also check it), config/resource discovery, and provider prestart reject it with the typed `profile_unavailable` error. The daemon reconciles the journal at boot **before** accepting traffic; `failed` ops surface in diagnostics and via the ops endpoints (`_dx.md`), and retry is explicit — a half-finalized profile is never selectable or servable.

Races covered by tests: rename-vs-create/delete, archive-vs-claim/trigger/spawn (single claim transaction — ADR-001), concurrent same-name create, install-vs-create, crash between apply and finalize.

### API Endpoints

Handlers land as `BaseHandlers` methods in `internal/api/core` (shared HTTP/UDS registration — no transport-duplicated validation): profiles CRUD + rename/archive/unarchive + delete; selection get/put; extension enablement get/put; scoped-read params (`profile`, `all_profiles`) on session/task/loop/automation/worktree/usage listings and the catalog stream. Status codes and payloads exactly as `_dx.md`; error codes from its table map to typed payloads in `marshalStructuredExecutionError`'s registry. Remote surface tier rejects **every profile-state write** — lifecycle mutations, `PUT /api/profiles/selection`, enablement writes — with `profile_remote_management_forbidden`; scoped and labeled-aggregate reads are all a remote tier gets (D12). Route registration mirrored by hand in both listeners (no parity test exists — path-map correction 13 — the tasks add one for the profile routes).

Plan and read projections (the frozen dialogs' backing — no TOCTOU, no client-side scans): `GET /api/profiles/{name}/rename-plan?new_name=…` · `GET /api/profiles/{name}/archive-plan` · `GET /api/profiles/{name}/delete-plan` return the revisioned Plan payloads from Core Interfaces; mutations carry `plan_revision` and fail `409 profile_plan_stale` when stale. `GET /api/profiles/selection` with no params returns the full workspace→profile map (Settings). Notification-preset enablement is a full surface: `GET /api/notifications/presets?profile=…` (effective state) and `PUT /api/notifications/presets/{name}/enablement {profile, enabled}`, with CLI verbs to match. Single-item reads accept `all_profiles=true` and return the owner-labeled resource — the S3 deep-link banner is this labeled aggregate-by-id lookup, never a client-side exception (the two read modes hold for gets too).

### Read-Scope Enforcement Matrix

The executable sweep contract — every A/A-inh read surface, its seam, its consumers, its canonical test. Phase 3 is complete when every row is wired and green; a surface absent from this table may not ship a listing.

| Surface | Seam (store/service) | Consumers | Test |
| --- | --- | --- | --- |
| sessions list/get + catalog stream | catalog queries + subscribe filter | web, CLI, API, native tools | IT-011, IT-023..025, E2E-011 |
| attention summary + badges + mutes | attention queries + `attention_workspace_mutes` | web menubar/dock, CLI | IT-013 |
| tasks + task_runs lists/gets | task service projections | web, CLI, API | IT-012 |
| loop runs + outputs | loop queries; outputs via owned run only | web, CLI | IT-012, IT-017 |
| automations (jobs/triggers/suggestions/runs) | automation queries | web, CLI | IT-012, IT-036 |
| worktrees (visible-everywhere, owner-tagged) | worktree queries | web, CLI | IT-015 |
| usage/token + cost | observe/token queries | web inspector, dashboards, CLI | IT-021 |
| observe overview/dashboard/inbox | observe engine (labeled aggregate) | web, CLI | IT-020 |
| network channels/threads/direct/work + timeline | network queries | web, CLI | IT-069 |
| network_audit_log + permission_log | audit queries (aggregate-capable) | CLI, system views | IT-069 |
| bridge instances/routes/deliveries/subscriptions | bridge queries | web, CLI | IT-014, IT-069 |
| notification cursors + preset effective state + dispatch/advance owner check | notifications queries + advancing tx predicate | Settings, CLI | IT-071, IT-078 |
| tool approval grants | grant queries + `tools.Scope` projection | approval surfaces, native tools | IT-013, UT-072 |
| event_summaries + dead_entities | reliability/observe queries | dashboards, doctor views | IT-020 |
| situation `/agent/context` bundle | situation assembler | agents | IT-022 |
| aggregate-by-id gets (deep links) | single-get with `all_profiles` owner label | web banners, CLI | IT-070 |
| memory catalog/search/recall (incl. FTS) | memory store queries, profile-tier scoped; **aggregate refused** (analysis/10) | agents, CLI, situation | IT-058, IT-074 |
| memory writes/consolidation/maintenance | tier-scoped ops; two profiles never merge | daemon background roles | IT-059 |
| memory caches + native-tool memory projections | profile in cache identity | native tools | IT-074 |
| list cursors (all paged surfaces above) | fingerprint includes profile predicate | everything paged | IT-026 |
| caches (prestart, bridge instance, web query keys) | ProfileID in key | runtime + web | UT-040/041, E2E-013 |

## Integration Points

None external. Everything integrates against in-repo seams (store, config, vault, extension pipeline, window manager, web SPA).

## Impact Analysis

| Component | Impact | Description / risk | Action |
| --- | --- | --- | --- |
| `internal/daemon` boot | modified | deletes `~/` auto-registration; ensures default profile + layout | phase 0/1 |
| `internal/session` catalog/stream | modified | server-side filter (workspace, then profile axes); fail-closed | phase 0/3 |
| 17 root tables + 2 PK identities | modified | `00078` (phase-0 disposition) + `00079` (default seed, stamps, PK rebuilds, ref rewrites); immutability + active-owner triggers | phases 0–1 |
| `internal/config` load/write | modified | 2 layers, scope renames, denylist validator | phase 4 |
| `internal/settings` | modified | scope kinds, provider/hook write targets, `general` split (persona defaults section) | phase 4 |
| `internal/vault` | modified | `profiles` namespace + containment + rename rewrite | phase 4 |
| `internal/resources` + `internal/extension` | modified | scope widening + grant re-derivation + enablement/placement/markers | phase 4/5 |
| `internal/skills`/agents/loops discovery | modified | 2 new roots; precedence rank decoupled from iota (persisted values stable) | phase 4 |
| `internal/tools`, `internal/providers`, `internal/situation`, `internal/observe`, `internal/notifications` | modified | ProfileID threading; canonical event key | phase 3/4 |
| `internal/cli` | modified | root `--profile`, `compozy profile` group, `--all-profiles`, config `--scope user\|profile\|workspace` | phase 2 |
| `web/` menubar, listings, settings, extension pages | modified | per `_uiux.md` S1–S13 | per capability slice (phases 2–6) |
| `packages/site` docs | modified | per-capability sections shipped with each slice; phase 7 stitches the user-first tutorial + precedence/file-locations pages + generated refs | phases 2–7 |
| `skills/compozy/` official skill | modified | profile verbs, selection rules, layer paths, error codes — per capability slice | phases 2–5 (+7 editorial) |

**Delete targets (no fallback, no compat shim, no placeholder):**

1. `internal/daemon/boot_runtime_foundation.go:176-200` `ensureDefaultWorkspace` + cascades (`:133-139` overlay special case, `:165` call, `:202-208` `operatorHomeDir`) — D3.
2. `internal/config/home.go:106-167` `ResolveOperatorHomeDir*` — re-audit; the `.compozy`-suffix fallback dies with its consumer.
3. Client-side catalog filtering in `web/src/systems/session/hooks/use-session-catalog-streams.ts` — replaced by server filter params.
4. Stored/CLI/API scope value `global` (config `WriteScope`, `settings.ScopeKind`, `resource_records.scope_kind`, `--scope global`) → `user`; memory scope `global` → `profile` (ADR-013). No dual values accepted anywhere.
5. `attention.muted_workspaces` global-array shape → typed per-profile rows (`attention_workspace_mutes`, Data Models; scope contract change target 4).
6. `providers`/`hooks` collections hard `ScopeGlobal` gates at the six call sites (`internal/settings/collections.go:102-175`) — replaced by profile-aware targets (sandboxes stay user-level).
7. The legacy `~/` workspace row + its registration path; its work rewritten to the `''` sentinel.
8. `~/.compozy/memory/` as a live path — contents move to `profiles/default/memory/`.
9. Pre-profiles session-catalog stream contract (no `profile` field) — payload replaced, not versioned.
10. `internal/store/workspacedb/` is explicitly **not** repurposed as a profile store (stays as-is; own cleanup out of scope).
11. `extensions.enabled` + `notification_presets.enabled` columns, their indexes, queries, DTO fields, and CLI/API behaviors — replaced by the per-profile enablement tables (rebuilds in `00079`; prior disabled state → `default`-profile exception rows).
12. Scalar `scopeRank` helper and the duplicate `scopesThrough` in the extension surfaces registry — replaced by the single lattice owner (Invariant 11).
13. `extension_env_bindings`' current `(extension_name, workspace_id, env_name)` identity — rebuilt with the profile dimension (Data Models).
14. The current `SessionCreationProfile` digest-input shape — replaced by the profile-id-bearing witness (Data Models); otherwise-identical sessions in different profiles must digest differently.

## Compozy Impact Audit

- **Native tools**: adds `compozy__profile_list` + `compozy__profile_current` — toolset registration, descriptors, input/output schemas (empty args → the typed results in `_dx.md`), regenerated schema digests, risk flags none (read-only), availability always-within-session, no capability gate; `tools.Scope` gains `ProfileID`, refreshing every builtin descriptor digest. Co-shipped with the phase-2/3 slices; asserted by session-derived scope tests (UT-072/073, IT-066). No other `compozy__*` IDs change (checked: `internal/tools/builtin_ids.go` inventory).
- **Extensibility and hooks**: manifest `[[profiles]]` + per-resource placement, per-profile enablement join, capability-grant re-derivation (D24 pins), hooks profile layer, bridge SDK cache key, MCP scope widening, config lifecycle (two layers + denylist) — full enumeration in the Extensibility Integration Plan below; extension-authoring/protocol docs update within each owning slice.
- **Workspace data isolation**: profile becomes the second scoping axis; propagation is traced end to end — CLI resolver → API params → `api/core` handlers → store ReadScope → SSE subscriber filters → web query keys/caches → events (`profile_id` correlation key) — enforced by the Read-Scope Enforcement Matrix and Safety Invariants 1–6, tested by IT-011..026, IT-069/070/074, and E2E-011/013.
- **Official Compozy skill**: `skills/compozy/` gains profile verbs, selection rules, layer paths, and error codes per capability slice (phases 2–5); phase 7 only stitches narrative — no public behavior waits for it.

## Extensibility Integration Plan

- **Manifest**: new `[[profiles]]` block (name, color/icon, `[profiles.defaults]`, `[[profiles.credentials]]`) + per-resource `profile` placement key — normalize/validate in the existing three-file chain; unknown/invalid names fail install (US-023.EC-5).
- **Kit publish**: placement filter (enablement AND placement) applied where surfaces project; eight publishable kinds share the widened `LegalScopes`.
- **Capability grants**: scalar `scopeRank` is deleted; one shared lattice owner (`meetScopeCeilings` + closure table) serves capability narrowing **and** the surface-registry expansion (the second `scopesThrough` implementation is a delete target), pinned by both canonical suites (D24); `ExtensionsConfig.Resources.MaxScope` accepts the new values.
- **Hooks**: profile config layer (D19); hook resources placeable like any kind.
- **Bridge SDKs**: `bridgesdk.InstanceCache` key gains profile (bound-secret cache separation).
- **MCP sidecars**: server scope gains profile value (`mcp-servers` collection C-default + A-override).
- **Official skill + protocol docs**: `skills/compozy/` and extension-authoring docs updated with placement/declared-profiles inside the owning capability slice (phase 5); phase 7 is editorial stitching only.

## Agent Manageability Plan

Exactly the `_dx.md` surface: `compozy profile` verb group (structured output on every verb, typed errors), root `--profile`/`COMPOZY_PROFILE`, `--all-profiles` listing option with owner-labeled JSONL frames + profile-resolution frame, `compozy config set --scope user|profile|workspace` with overridden-write feedback, `compozy secret`/`provider inspect` resolution visibility, extension enablement verbs, HTTP/UDS parity on all routes, native tools `compozy__profile_list`/`compozy__profile_current` (session-derived, read-only). Machine commands ignore profile context by contract. Agents can fully operate lifecycle through CLI/API from inside sessions except re-aiming their own binding (D9).

## Config Lifecycle

- **Keys**: none added. **Layers**: `~/.compozy/profiles/<p>/config.toml` and `<ws>/.compozy/profiles/<p>/config.toml` join the merge order (`loadWithHome` — the only overlay site); repo profile layer is read-only to the CLI (hand-authored, committed). MCP sidecars follow the same two layers (`profiles/<p>/mcp.json` on both sides), merged in their layer's slot with the same write-target rule (Data Models).
- **Write targets**: context-owned default (user file under `default`, profile file otherwise); `--scope user|profile|workspace`; denylist (list above) rejected fail-closed with allowed-prefix guidance; "saved but not applied — <layer> wins" on overridden writes (OkOverridden contract, D16).
- **Validation**: profile-overlay validator mirrors the workspace one; orphan profile config files (no matching profile) are diagnostics, never half-applied.
- **Docs**: `configuration/index.mdx` precedence table, `file-locations.mdx` layout, new profiles docs; generated CLI docs regen.
- **Tests**: load/merge order, write-target arms, denylist, overridden feedback, orphan diagnostics; `config_apply_records` entries name the layer (D26 — single timeline preserved).

## Testing Approach

- **Unit** (per package, `-race`): profile domain state machine (lifecycle, grammar, reserved names), resolver chain, selection store sweep, overlay merge/write targets/denylist, vault ref containment, placement/enablement/marker logic, scope-widening validators, grant-ordering pin (D24).
- **Integration** (`+integration`): migration suites — fresh/reopen/ahead/integrity/equivalence for `00078` (phase-0 disposition), `00079` (default seed, stamps, PK rebuilds, ref rewrites), and the memory-stream migration (scope values + durable move op); fail-closed store reads (both modes); listcursor fingerprint isolation; stream filter (subscribe-level, replay included); prestart cache key separation; lifecycle protocol (plans, journal, forward-only recovery — rename proves id-keyed selections stay untouched).
- **E2E runtime** (Go harness): golden path CLI journeys from `_dx.md` verbatim (create→exec→scoped list→aggregate→use→current), lifecycle errors, extension install with declared profile (mock kit), config write feedback; UDS/HTTP parity for profile routes.
- **E2E web** (Playwright): `_uiux.md` journeys — switcher/create/switch, aggregate labels + chip + toast, settings lifecycle dialogs, extension needs-setup, workspace hint, per-workspace restore.
- Placement: profile resolution asserts join `internal/cli/workspace_test.go` + `session_test.go` suites (path-map correction 14); no new standalone regression files where a canonical suite owns the invariant. Concrete cases live in `_tests.md`.

## Development Sequencing

### Build Order

**Vertical-slice rule (no partial surfaces):** a capability is phase-complete only when everything it touches ships together — contract spec file → shared `api/core` handler → HTTP **and** UDS registration → CLI verb/client → OpenAPI + generated types (`make codegen`) → **its affected Web paths** → **finished docs and official-skill content for that capability** → behavioral tests (incl. route parity). Later phases add new capabilities; they never finish earlier ones.

- **Phase 0 — Foundation fixes** (D3+D4): migration `00078` per the disposition table (nullable rebuilds, home rewrite, `~/` row delete) + non-registrable guard; server-side catalog stream/list filtering (workspace axis) with fail-closed subscribe — itself a contract slice (params, codegen, parity, web client switched off client-side filtering). Gate: seeded disposition equivalence suite + leak tests.
- **Phase 1 — Profile domain + schema**: `internal/profile` + lifecycle operation protocol + boot reconciliation; `05_profiles.sql` + `00079` (catalog, selections, stamps, PK rebuilds, stored scope/ref rewrites, join tables, mutes, triggers); memory-stream migration (scope values + crash-safe dir move); boot default-profile ensure. Gate: migration equivalence suites + crash-recovery test (IT-072/073).
- **Phase 2 — Lifecycle & selection capability**: resolver (flag/env/remembered/default), `compozy profile` verbs backed by the plan endpoints, profiles/selection/plan routes on both transports, typed errors, resolution frames — **plus its web surfaces** (S1 switcher, S4 Settings page, S5 `SymbolPicker` in `packages/ui` with story+test, S6/S7 dialogs) and its finished docs section + official-skill entries. Gate: CLI suites + IT-062/063 + E2E-013/014/016/017.
- **Phase 3 — Scoping & aggregate capability**: the Read-Scope Enforcement Matrix row by row (ReadScope threading, stream profile axis, `tools.Scope.ProfileID`, situation, cursors, caches) **plus the aggregate UI** (S2 chip/toast, S3 labels/banners/empty states, S10 usage dimension, S11 globe interplay) and its docs/skill content. Gate: every matrix row green + E2E-015/018/019/021.
- **Phase 4 — Layers capability**: resource roots + scope widening + grant re-derivation (D24 pins); config layers + write targets + denylist + `--scope` values; vault namespace + provider cache key + secret CLI; `general` split; the memory read-path matrix rows — **plus their Settings/shadow-inspect UI** and docs/skill content. Gate: layer/shadow suites + D24 pins + IT-058/059/074 + E2E-006/007.
- **Phase 5 — Extensions & presets capability**: enablement join + routes/verbs, placement filter, declared profiles + markers + install summary, per-profile secret bindings, notification-preset enablement — **plus S8/S9/S13** and docs/skill content. Gate: extension e2e with mock kit + IT-071 + E2E-022/023/026.
- **Phase 6 — Per-profile state & polish**: S12 desktops partition (clientstate keying), cross-surface a11y sweep, remaining browser journeys (E2E-020/024/025). Gate: web build + full Playwright set.
- **Phase 7 — Editorial synthesis & QA prep**: site IA stitching of the per-slice docs into the user-first tutorial, glossary entry ("Profile"), QA scenario flags, final generated-reference sweep. **No public behavior waits for this phase.** Gate: `make gate-full` (workstream close).

### Technical Dependencies

Phase 0 precedes everything (enforcement point + delete site). Phase 1 precedes 2–6. Phases 4 and 5 depend on 1; 5 depends on 4 (vault/placement). 6 depends on 2–3 (API + scoping) and consumes 4–5 surfaces. No external dependencies.

## Monitoring and Observability

- Canonical events: `profile.created|renamed|identity_updated|archived|unarchived|deleted`, `profile.selection_changed`, `extension.profile_created`, and the profile-keyed enablement mutations `extension.enablement_changed` / `notification_preset.enablement_changed` — with `profile_id`, `profile_name`, `actor_kind`.
- Lifecycle-protocol events: `profile.lifecycle_op_recovered` / `profile.lifecycle_op_failed` (operation id, kind, step, outcome) and a `profile.plan_stale` rejection event — never carrying secret refs or raw tokens.
- `profile_id` joins the canonical correlation keys on session/task/loop/automation lifecycle events; the coverage-matrix test extends to the new lifecycle paths.
- Doctor/diagnostics stay machine-wide and say so; support bundles remain machine-scoped (explicitly labeled — they contain every profile).

## Technical Considerations

### Key Decisions

- **Precedence rank decoupled from iota**: new resource source values append to the enum; an explicit rank function orders them (no renumbering of persisted comparisons — path-map §6 hazard).
- **Enablement rows only for exceptions** (absent = enabled): fewer rows, trivially correct default for new profiles.
- **`workspace_profile` scope_id composite** (`<ws>@pf:<name>`): name-bound by design (dormancy semantics), single validated segment grammar.
- **Vault refs carry the profile *name*, rename rewrites them**: readable refs (D7 ergonomics); the rewrite commits in the rename's apply transaction (Lifecycle Operation Protocol step 2), and selections — id-keyed — are untouched (US-003.AC-1 "credentials follow").
- **`work_items` counts stamped roots only**: the number on `profile list`, plans, Settings, and aggregate labels = rows the profile owns across the 17 creation roots — descendants excluded (they inherit), worktrees included (ownership ≠ visibility). One query definition reused everywhere; nothing double-counts.
- **Selection is daemon-owned** (table, not browser storage): CLI and web share it; web caches for instant render (ADR-014).
- **`loop_runs.origin_creation_profile_ref` naming trap**: the new column is `profile_id`; migration text and mappers must never touch the creation-profile digest field (path-map correction 16).

### Known Risks

- **Enforcement sweep breadth** (highest): a missed A-class read leaks. Mitigation: scope contract (analysis/10) is the checklist; fail-closed defaults; phase-3 integration tests enumerate every listing surface; `situation` assembler test.
- **PK rebuild migrations** (`token_usage_daily`, `notification_cursors`): rebuild + backfill under test with equivalence suites; append-only identity respected.
- **Grant-ordering regression**: D24 pinned tests ship in the same change as the enum.
- **Route parity drift**: hand-mirrored listeners; profile routes ship with a parity test (first of its kind — path-map correction 13).
- **Web cache bleed**: profile switch must bump stream generations and query-key namespaces; Playwright asserts no stale rows after switch.

## Safety Invariants

1. Every stamped-root INSERT carries a valid `profile_id` **whose profile is active** — enforced by the per-root creation trigger inside the same write transaction (typed `profile_archived`); store methods never default the stamp — boundaries resolve explicitly (API absent-param = `default` at the handler, CLI via the resolver, agents via session identity).
2. Exactly two read modes exist; aggregate is explicit (`all_profiles`), always owner-labeled; no third mode, no silent widening.
3. Fail-closed everywhere: unresolved/invalid profile context yields typed error or empty result — never unfiltered rows (streams included, replay included).
4. `profile_id` is immutable on every root (ABORT triggers); no transfer path exists in v1.
5. List cursors fingerprint the profile predicate; a cursor never replays across profiles.
6. Agent-facing surfaces derive the profile from daemon-validated session identity; a caller-supplied acting profile that differs from the session binding fails with the typed `profile_session_conflict` — rejected, never silently ignored (D9); explicit aggregate reads remain the only widening.
7. Delete requires zero owned work across all 17 roots in one transaction; profile-tier **memory and desktop partitions are not work** — they are enumerated in the plan (`RemovalSummary.MemoryEntries` / `DesktopPartitions`) and removed by apply/finalize (memory rows + directory; clientstate `(workspace, profile)` partitions purged) — as are the profile's MCP sidecar server entries (`RemovalSummary.MCPServers`, removed with the profile folder) and its loops (`Loops`); the removal catalog is **total**, so the preview exactly matches the applied removals and nothing profile-owned survives. Archive requires no running sessions and no leased/running task runs, freezes the profile's queued runs, and pauses automations — all inside one immediate transaction (ADR-001).
8. Extension declared-profile creation is create-once per (instance, name); existing names bind, never seed; updates never mutate; operator archive/delete is never reverted by any extension path.
9. Profile vault refs resolve only under owner-prefix containment for the acting profile; `env:` refs are a validation error for profile scope.
10. Delete sweeps `profile_selections` (FK-backed); **archive leaves rows in place** — resolution then falls back to `default` with `archived_remembered_fallback` provenance; a remembered entry never yields an archived acting profile.
11. The ceiling lattice (downward closures + pairwise meets) lives in **one owner** (`meetScopeCeilings` + closure table) consumed by both capability narrowing and the surface-registry expansion, pinned by tests in the same change that widens the enum (D24); the scalar `scopeRank` helper is **deleted** — a total order cannot express incomparable axes.
12. Network delivery is never predicated on profile (tag ≠ block); cross-profile conversation is regression-tested.
13. Scheduler/budget/spawn primitives remain profile-blind for fairness (per-machine, D12); `ClaimNextRun` keeps its signature and stays the sole claimer — its single claim query additionally requires the owning profile to be active (no peer gate, no second queue; L-003/L-005).
14. Lifecycle mutations follow prepare/apply/finalize: row state and the operation journal commit atomically after plan-revision validation; pre-commit failures roll the transaction back (no filesystem touched); post-commit recovery is **forward-only** from the journaled steps (idempotent, boot-reconciled); repo files change only as reported pending edits with per-workspace consent; extension files never.
15. A plan-backed mutation never commits against a stale plan (`profile_plan_stale`); prepare is strictly read-only.
16. Selection rows are id-keyed: rename never touches them; the delete sweep is their only lifecycle mutation (archive preserves them as fallback provenance).
17. A profile with a pending lifecycle operation is unavailable everywhere (selection, creation, discovery, provider start — typed `profile_unavailable`); the boot order runs journal reconciliation before the daemon accepts traffic.
18. Notification dispatch uses one **owner-active permit** shared with archive (no queue, no event tailing), durably owned by the `notification_delivery_permits` table (Data Models): dispatch acquires the permit (row INSERT in an immediate transaction that verifies the owner is active) *before* the external send, holds it through acknowledgement and cursor advancement, then clears it in the advancing transaction; archive's apply transaction refuses while any permit row exists for the owner — typed, retryable `profile_deliveries_in_flight` — and, once committed, no new permit is grantable. The notifications service owns crash recovery: a surviving permit row replays by deterministic delivery id (the existing cursor contract: same sequence + same delivery id only), so nothing double-delivers on unarchive.
19. Pending lifecycle operations reserve their names and paths (Data Models); a create/rename targeting a reserved name fails with `profile_name_taken` naming the pending operation — boot recovery therefore never mutates a profile it does not own.

## File References

Read-first index; tasks copy their subsets. Verified anchors: `analysis/11_arch-path-map.md` (wins over older slices on drift).

### Repo Files

- `internal/cli/workspace_resolution.go:15-24,84-146,338-393` — resolution ladder, typed errors, context record; the profile resolver's template and join point.
- `internal/cli/root.go:63-106,166-191,273-330` — deps injection, the only persistent-flag site, typed-error registry.
- `internal/cli/format.go:63-175` — output bundle contract + resolution JSONL frame.
- `internal/config/home.go:71-98,217-283` — HomePaths derivation + layout creation (profile dirs join here).
- `internal/config/config_load.go:39-52,137-174` — the only overlay merge site; `WithProfile` option lands here.
- `internal/config/persistence.go:21-53,87-148` + `write_scope_policy.go:9-25` + `workspace_overlay.go:41-55` — write scopes, target mapper, denylist validators.
- `internal/settings/models.go:15-104` + `collections.go:26-177` + `sections_daemon.go:10-93` + `collection_provider_write.go:311-325` — scope kinds, hard-global gates to cut, `general` split site, provider vault prefix.
- `internal/store/globaldb/schema/definitions/00_core.sql`, `20_sessions.sql:89-204`, `33_extensions.sql`, `34_resources.sql:1-19`, `37_notifications.sql`, `50_loops.sql:509-585`, `70_tasks.sql`, `72_task_runs.sql` — stamped roots + scope idioms + house templates.
- `internal/store/globaldb/queries/session_core.sql:56-88`, `task_core.sql:1-18`, `loop_core.sql:1-27`, `automation_core.sql:77-127`, `bridge_core.sql:1-40`, `worktrees.sql:1-25`, `observe_overview.sql:2-20` — the stamping INSERT/UPSERT sites.
- `internal/session/session_catalog_stream.go:33-96` + `internal/api/core/session_catalog_stream.go:15-61` + `internal/session/catalog_page.go:41-58` — the D4 fix site end-to-end.
- `internal/resources/types.go:61-120` + `projector.go:16-116` + `internal/extension/capability_resource_policy.go:115-136` + `surfaces/registry.go:42-100` — scope widening + grant ordering + publish surfaces.
- `internal/extension/manifest.go:88-130` + `manifest_normalize.go` + `manifest_validate.go` + `install_managed.go:31-60` + `internal/config/home_extension_data.go:15-54` — manifest chain, install root, `@pf-` segment template.
- `internal/vault/types.go:29-48` + `mcp_refs.go:11-125` + `extension_refs.go:6-19` — ref grammar + containment templates.
- `internal/providers/probe_env.go:81-88` + `prestart_cache.go:187-205` + `internal/providerenv/env.go:227-245` — credential cache key + id grammar constraints.
- `internal/skills/types.go:95-111` + `registry_source.go:18-65` + `internal/config/agent.go:154-211` + `internal/workspace/scanner.go:281-315` — precedence enum hazard + discovery roots + collision rule.
- `internal/workspace/workspace.go:14-80` + `resolver_crud.go:13-122` — sentinel-error and two-phase CRUD templates for profile lifecycle.
- `internal/daemon/boot_runtime_foundation.go:133-208` — the phase-0 delete site; `internal/daemon/window_manager_repository.go:17-28` + `window_manager_boot.go:31-40` — desktops clientstate partition.
- `internal/tools/tool.go:284-291`, `internal/agentidentity/identity.go:1-25`, `internal/situation/service.go:17-35`, `internal/listcursor/cursor.go:1-25`, `internal/notifications/scope.go:19-45`, `internal/clientstate/contract.go:1-93` — threading + identity + leak-visibility + cursor + selection-store seams.
- `web/src/systems/os/components/os-menubar.tsx:21-226` + `desktop-menubar.tsx:183-250` + `global-scope-toggle.tsx:22-83` + `menubar/workspace-menu.tsx:27-50` — menubar wiring for S1/S11.
- `web/src/systems/session/hooks/use-session-catalog-streams.ts:40-65` + `session-catalog-streams-store.ts:34-50` + `web/src/systems/workspace/lib/query-keys.ts:1-19` + `stores/active-workspace-store.ts:14-135` — stream/query/selection client seams.
- `packages/ui/src/exports/foundation.ts:27-83` + `listing.ts:53-60` + `packages/ui/src/index.ts:1-13` — reuse inventory for switcher/picker.
- `internal/api/spec/` + `Makefile:65-69` — contract-codegen co-ship path.

### Competitor References

- `.resources/hermes/hermes_cli/profiles.py:1-58` — profile anatomy/lifecycle verbs; `default`-asymmetry to reject (uniform `default` here).
- `.resources/hermes/docs/profile-routing.md:9-117` — the route table explicitly deferred (ADR-011) and its enforcement posture (fail-closed, no default fallback) which phase 3 ports.
- `.resources/hermes/docs/design/profile-builder.md:6-120` — creation-UX conclusions (dashboard-native; identity-first) behind S4/S5.
- `.resources/codex/` profile-v2 sources per `analysis/09` — sibling-config write-target + `OkOverridden` feedback + name newtype + denylist footgun (D16 lineage).

### Design and Analysis Sources

- `analysis/10_analysis_profile-scope-inventory.md` — the authoritative scope contract (stamped roots, classes, delete targets); Part II's checklist.
- `analysis/11_arch-path-map.md` — verified anchors + 16 corrections; wins on drift.
- `analysis/01..09` + `analysis/summary.md` — mechanisms and superseded strategy verdicts (supersession note in summary Round 2).
- `DECISIONS.md` (D1–D27 as amended) + `adrs/adr-001..015.md` — binding decisions.
- Artboards (to be produced by the design pass): `docs/design/opendesign/profiles/` per `_uiux.md`.

## Assumptions and Defaults

- Profile name grammar `^[a-z][a-z0-9-]{0,31}$`; reserved `default`, `all`, `global`; uniqueness across active+archived.
- Absent API `profile` param = `default`; absent CLI selection = resolution chain; store layer never defaults.
- New profile defaults: extensions enabled, presets default, clean layout, user credentials, no overrides (extension-declared profiles start from their seed).
- Pre-profiles data (work, usage, memory global tier, selections) belongs to `default` after migration; no data is deleted except the `~/` workspace row (work preserved via `''` sentinel).
- Identity color free-pick + suggested palette; AA foreground computed; icon set is the product's bundled set (implementation detail of `SymbolPicker`).
- The `default` profile id is the fixed zero ULID `00000000000000000000000000`, seeded by migration `00079` before any backfill; boot only verifies it and ensures its directory layout idempotently (ADR-012).
- Archived profiles keep names reserved; sticky fallback is silent (switcher shows `default`).
- Worktree placement-root collision across profiles (same workspace) is acceptable in v1 — worktrees are visible everywhere by rule; path uniqueness already includes the worktree name.
- Support bundles/logs remain machine-wide; copy labels them so.
- No cap on profile count; switcher designed for 3–6.

## Architecture Decision Records

- [ADR-001](adrs/adr-001.md) — Archive instead of delete-with-work or transfer.
- [ADR-002](adrs/adr-002.md) — Extension-declared profiles auto-create at install (create-once markers).
- [ADR-003](adrs/adr-003.md) — Selection remembered per workspace (real profiles only).
- [ADR-004](adrs/adr-004.md) — Core ownership model: fixed workspaces, stamped work, uniform default, one runtime.
- [ADR-005](adrs/adr-005.md) — "All profiles" switcher state; create-in-default with fixed chip.
- [ADR-006](adrs/adr-006.md) — Four additive resource layers bound by profile name (as amended).
- [ADR-007](adrs/adr-007.md) — Foundation fixes land inside this spec as phase 0.
- [ADR-008](adrs/adr-008.md) — Naming: bare "Profile"; compounds unrenamed; scope-word mapping.
- [ADR-009](adrs/adr-009.md) — User credentials by default; per-profile vault overrides.
- [ADR-010](adrs/adr-010.md) — Organization, not access control.
- [ADR-011](adrs/adr-011.md) — No inbound routing table in v1; entry points own deliveries.
- [ADR-012](adrs/adr-012.md) — Identity: stable ULID stamp + unique name handle; rename rewrites daemon-owned name-bearing artifacts.
- [ADR-013](adrs/adr-013.md) — Scope vocabulary hard cut: stored `global` → `user` (and memory tier → `profile`).
- [ADR-014](adrs/adr-014.md) — Remembered selection is daemon-owned state.
- [ADR-015](adrs/adr-015.md) — Stamp 17 roots, inherit the rest; two read modes enforced at the store layer.
