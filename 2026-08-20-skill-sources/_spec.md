# Spec: Skill Sources

Configurable skill source folder patterns — CompozyOS absorbs skills from ecosystem folder conventions the user selects, applies changes live, keeps absorbed skills fully manageable, and can expose its own skills back to external tools.

---

# Part I — Product

## Overview

**Problem.** CompozyOS only loads skills from its own folders (`.compozy/skills` in a workspace, the CompozyOS home skills folder globally). Meanwhile the agent-tool ecosystem converged on folder conventions for skills: a universal cross-tool location (`.agents/skills` in a project, `~/.agents/skills` for the user) read natively by most agent CLIs, and per-tool locations such as Claude Code's `.claude/skills` and `~/.claude/skills`. Operators who work with several tools already own skill libraries in those folders — and CompozyOS cannot see any of them. The reverse is also true: skills authored or installed in CompozyOS are invisible to every other tool.

**Who it is for.** Operators running CompozyOS alongside other agent tools; teams that commit a shared skill folder to their repositories; skill authors who want one canonical copy readable everywhere; and managing agents that operate CompozyOS through structured surfaces.

**Why it is valuable.** Zero-setup absorption of the user's existing skill library (the universal folders load by default), explicit user control over which conventions are absorbed (per installation and per workspace), and two-way interop: CompozyOS both reads the ecosystem's folders and can expose its own skills into them — one canonical body per skill, no copies to drift.

## Goals

After this ships:

1. Skills in the universal folders (`.agents/skills` in the workspace, `~/.agents/skills` globally) load by default, with no configuration.
2. The user selects which folder conventions CompozyOS scans — curated source presets plus arbitrary custom directories — and can override that selection per workspace.
3. Source changes apply live: toggling a source makes its skills appear or disappear within seconds, without restarting.
4. Absorbed skills are full citizens: they appear in the session `/` picker, are invocable, can be enabled/disabled, participate in name-collision resolution with a visible audit trail, and carry their origin wherever skills are listed or chosen.
5. A session's context never receives the same skill twice: skills the session's provider already loads natively from its own folders are omitted from injected context — while remaining visible, manageable, and explicitly invocable.
6. A skill with a real on-disk home (authored, installed, or custom) can be exposed into enabled ecosystem folders as a per-skill link, and lifecycle operations (remove, update) never leave broken links behind.
7. Managing agents can inspect sources (enabled state, resolved locations, existence, per-source counts, truncation) and change the configuration through structured command-line and programmatic surfaces with deterministic errors — no web UI required.
8. Scan limits and name collisions are always visible in product surfaces, never silent.

## User Stories

Full catalog: [_user_stories.md](_user_stories.md).

- US-001–US-003 — Source configuration: defaults, preset toggling with live apply, custom directories.
- US-004–US-005 — Workspace behavior: committed repo skills, per-workspace override with inheritance.
- US-006–US-007 — Discovery semantics: deterministic collision resolution, symlinked installs appearing once.
- US-008–US-009 — Session usage: `/` picker coverage with origin labels, duplicate-context avoidance.
- US-010–US-011 — Expose: create-time expose, expose/unexpose lifecycle with cleanup guarantees.
- US-012 — Agent management: structured inspect/configure surfaces at both scopes.
- US-013 — Settings UI: the sources section in Settings > Skills.
- US-014 — Diagnostics: visible truncation, collisions, and skipped links.

## Core Features

1. **Source presets.** A curated, daemon-known table of folder conventions, each expanding to a workspace-level and/or global-level location. v1 table: `compozy` (CompozyOS's own folders — always on, not configurable), `agents` (universal convention — enabled by default), `claude` (Claude Code convention — available, off by default). New presets are additive curated changes.
2. **Custom source directories.** User-registered directories scanned as skills-only sources, for layouts no preset covers.
3. **Workspace override.** A workspace can replace the installation-wide source selection (presets and custom lists independently); absent an override it inherits, and surfaces state which is happening.
4. **Live application.** Source configuration changes take effect on the running system within seconds; surfaces report the apply semantics truthfully.
5. **Full-citizen absorption.** Absorbed skills enter the same catalog as native ones: same verification on load, same enable/disable, same collision/shadow audit, same session picker and invocation path, same budgets.
6. **Duplicate-context avoidance.** Each preset knows which session providers read its folders natively; the injected context omits skills the provider already has, keeping everything else (picker, management, explicit invocation) intact, with every omission observable in diagnostics.
7. **Skill expose.** An eligible skill (one with a physical directory) can be linked into any enabled source the user names — at creation time, via a dedicated command, or from the skill's web surface. Removal cleans up links; updates preserve them; foreign links are never touched; failures are deterministic (no silent copy fallback).
8. **Origin visibility.** Skills from non-Compozy sources carry a discreet origin label in the `/` picker, skills lists, settings, and structured outputs.
9. **Structured management.** Every operation above is available to agents via command-line and programmatic surfaces with parity to the web, structured outputs, and deterministic errors.
10. **Diagnostics.** Truncated roots, unclaimable names (collisions), and skipped links (dangling, escaping, cyclic) are reported per source in management surfaces.

Feature interactions: presets/custom (1, 2) feed the catalog that absorption (5) resolves; the override (3) selects which lists apply per workspace; live application (4) governs 1–3; avoidance (6) reads the preset table's provider knowledge; expose (7) writes into folders that 1–2 declared trusted; visibility (8) and diagnostics (10) render what 5–7 record; management (9) spans all of it.

## Business Rules

**Selection and validation**

1. The `compozy` source is always active and never appears in the configurable selection; an empty selection is valid and means "Compozy folders only".
2. An unknown preset name is a validation error naming the valid slugs and the closest match; it is never interpreted as a path.
3. Custom entries accept absolute and home-relative paths at global scope; workspace scope additionally accepts workspace-relative paths. A duplicate of an already-active root (same resolved location) is rejected naming the existing source.
4. Workspace override replaces the whole list per key (presets and custom independently); no override means inherit. Removing the override restores inheritance.

**Discovery and resolution**

5. A skill is a directory containing a `SKILL.md` definition; the definition's name identifies it. Definitions missing a name are rejected per-skill with a diagnostic; the rest of the root still loads.
6. Precedence: workspace-level sources beat global-level sources; within the same level, CompozyOS's own folders win ties. Every collision records the losing definitions in an inspectable audit trail — the winner is never silent.
7. One physical skill appears exactly once, no matter how many links point at it (identity is the resolved real location). First-level links inside a source root are followed; links that escape trusted locations, dangle, or cycle are skipped with a per-entry diagnostic.
8. Ecosystem-standard definition fields that CompozyOS does not act on are accepted without warnings; they remain inert. Truly unknown fields still warn.
9. Every non-bundled skill passes content verification on every load, regardless of which source it came from; critical findings block that skill.
10. Per-root scan limits hold; exceeding them flags the source as truncated in every management surface.

**Session behavior**

11. When a session's provider natively reads an enabled source's folders, skills whose winning copy lives there are omitted from that session's injected context. The omission never removes the skill from the picker, lists, or management, and an explicit invocation always delivers the skill. Unknown provider → nothing is omitted.
12. Skills from every enabled source are invocable in sessions via `/`, with the same verification, rejection, and budget rules as native skills; same-named skills from different sources stay distinguishable and invocable through qualified forms.

**Expose**

13. Only skills with a physical directory are exposable; bundled skills are not (they have no on-disk home) and the error says so.
14. Expose targets must be enabled sources; the link lands at the skill's scope level (workspace skill → workspace-level folder, global skill → global-level folder). A name conflict at the target is an error; re-exposing an existing healthy link is idempotent.
15. Removing a skill removes every link CompozyOS created for it; updating a skill in place preserves links. CompozyOS never removes a link it did not create, and never falls back to copying content when linking fails.

**Access**

16. Source configuration is installation and workspace policy: editable at those scopes by operators and managing agents; read-only at agent scope.

## User Experience

**Journeys**

1. *Multi-tool operator, first contact*: installs CompozyOS on a machine with an existing `~/.agents/skills` library → opens a session, types `/`, and their skills are already there. Later they open Settings > Skills, see the sources section with per-source counts, and switch the Claude convention on; its skills appear in the picker within seconds, labeled by origin.
2. *Team member*: clones a repo whose `.agents/skills` is committed → opens it as a workspace → the team's skills are available in every session there, and only there. The repo lead adds a workspace override enabling one extra convention for that project only.
3. *Skill author*: creates a skill, chooses to expose it to the universal folder → their other tools list it immediately; editing the canonical copy propagates everywhere. Removing the skill later leaves no broken links.
4. *Managing agent*: reads the active sources with counts and existence states, notices a truncated custom root, replaces it with a narrower directory, and confirms the counts — all through structured commands.

**Discoverability.** The sources section lives inside the existing Settings > Skills page; per-source counts and states make the effect of each toggle legible; skill lists and the picker carry origin labels; documentation covers the folder conventions and the agent path.

**Accessibility.** The section follows the settings surface's existing interaction patterns (toggle rows, labeled controls, keyboard navigation); the UI change map (`_uiux.md`) carries the screen-level detail.

## High-Level Technical Constraints

- Integrates into the existing skill catalog, marketplace, settings, and session picker — one discovery system, not a parallel one; absorbed skills obey every existing skill governance rule (verification, enable/disable, budgets).
- Security: content verification applies to every absorbed skill on every load; directory and link resolution never follows an escape outside user-trusted locations; CompozyOS writes into non-Compozy folders only for user-requested expose links.
- Perceived performance: a saved source change is visible on every surface within two refresh intervals; scan limits cap pathological directories with visible truncation.
- Privacy: user directories are read only when the corresponding source is enabled.
- Agent/operator manageability: every capability here is operable through structured command-line and programmatic surfaces with web parity (inspect, configure, expose, diagnose) and deterministic errors.
- Extension ecosystem: extensions keep contributing skills through their existing resource mechanism, unaffected; the preset table is closed to extension registration in v1.

## Non-Goals (Out of Scope)

- **Remote skill installation or sync** — no fetching from external repositories, no lockfiles, no update checks against foreign sources. Remote acquisition stays exclusive to the Compozy marketplace; output of external installers is picked up by scanning (ADR-014).
- **Agent-definition discovery from new roots** — this feature covers skills only; agent definitions keep their current homes (ADR-014).
- **Extension-registered or user-defined presets** — the preset table is curated code; custom directories cover bespoke layouts (ADR-014).
- **Content mirroring into per-tool folders** — the only write into non-Compozy sources is the user-requested expose link; no copies, no background sync (ADR-014).
- **OpenClaw's bare project `skills/` folder** — permanently excluded from scanning; it collides with ordinary repository folders (this repo's own `skills/` included) (ADR-014).
- **Other tools' plugin/marketplace namespaces** — only their standard skill folders are read; plugin-scoped skills (e.g. `plugin:skill` forms) are out.
- **Honoring foreign frontmatter semantics** — ecosystem fields are recognized to keep diagnostics clean, never executed (ADR-008).

## Open Questions

None — every product decision reached in the grill rounds is recorded as an ADR (ADR-001…ADR-014; see Architecture Decision Records in Part II).

---

# Part II — Technical

## Executive Summary

The skills discovery pipeline gains configurable roots: a daemon-owned preset table (`compozy` implicit, `agents` default-on, `claude` off) plus free-form custom directories, resolved per scope into typed root specs that feed the three existing scan paths (workspace resolver, registry global load, registry workspace load). The existing six-tier precedence and name-keyed shadow machinery absorb the new roots unchanged — each tier simply owns a list of roots instead of one. Changes apply live through the watcher's roots provider plus explicit cache invalidation and resource republication. Session context stays lean via a provider-aware injection filter in the harness policy (suppress skills the session's provider loads natively; catalog/picker/API stay unfiltered). Interop back out is the expose primitive: per-skill symlinks from enabled preset roots to the canonical Compozy-owned directory, with **persisted ownership records** in the new `skill_exposures` side table (one append-only migration) and link health reconciled live from the filesystem (ADR-015). The wire contract grows additively (settings `sources[]` read model, `origin`/`exposures` on skill payloads, expose endpoints).

Primary trade-offs: replace-semantics workspace override (simple, matches repo overlay standard) over granular add/remove; session-resolved native-root suppression (deterministic, testable, fail-open) over ACP-driven dedup (impossible — the protocol carries no source info); persisted ownership records + filesystem-reconciled health over both pure-filesystem inference (cannot prove ownership or remember deleted links) and record-as-truth (drifts from the real filesystem).

## MVP Boundary

MVP = everything in this spec: config keys + preset table, three scan paths on resolved root lists, first-level symlink following with realpath dedup, live apply, workspace override (CLI + API + web), injection suppression, expose (verb + create flag + web panel), `skill sources` verb, settings section, picker origin chips, diagnostics (truncation/collision/skipped links), docs + official-skill updates, QA tail. Post-MVP (deferred, additive): additional presets (`codex`, `hermes`, `openclaw`, `cursor`, `opencode`), ACP advertised-command confirmation signal, custom sources as expose targets. Out of scope permanently (Part I Non-Goals): remote installer/sync, agent-definition discovery, extension-registered presets, content mirroring.

## Developer Experience

- [Developer experience contract](_dx.md) — golden path, CLI (`skill sources`, `skill expose|unexpose`, `create --expose`, `config set/get/unset` with the two new keys, `skill list/info/where` origin columns), HTTP/UDS (settings envelope + expose routes + skill payload fields), `config.toml` block, native-tool note, deterministic error table.
- [UI change map](_uiux.md) — S1 settings sources section, S2 picker origin chips, S3 skills catalog/detail expose panel; component plan (no new `@compozy/ui` primitives).

## System Architecture

| Component | Responsibility |
|---|---|
| **Preset table** (`internal/config`) | Static data: slug, label, workspace-relative path, global path, native readers, flags. Validation (unknown slug, did-you-mean), custom-source auto-slug (basename, deterministic collision suffix). |
| **Root resolution** (`internal/config`) | `SkillsConfig` × scope × home paths × workspace root → ordered `[]SkillRootSpec{Dir, SourceSlug, Scope}`. Single authority for "which directories, in which order, attributed to which source". |
| **Scanner** (`internal/skillscan`) | Walks each root; follows first-level symlinks with containment check + realpath identity; reports per-entry skips and per-root truncation. |
| **Registry** (`internal/skills`) | Loads bundled + resolved global roots (was: single `UserSkillsDir`); workspace load consumes resolved specs (was: hardcoded joins); realpath dedup across roots; tier mapping slug→`SkillSource`; per-root contribution stats for the read model. |
| **Live apply** (`internal/daemon` + `internal/skills`) | Single apply coordinator owns a monotonic config generation; watcher roots provider re-derives from current config; apply triggers refresh, workspace-cache invalidation, and resource republication via `DiscoverGlobal`/`DiscoverWorkspace` — every asynchronous commit point (registry swap, resource publication, diagnostics replacement, session-command revision broadcast) compares the current generation immediately before committing and discards stale work (B-009). |
| **Injection policy** (`internal/daemon` harness) | Session-resolved native-root filter (B-003): at session start the daemon resolves the provider's **actual** native skill roots — canonical provider id × per-root preset knowledge × the session's effective env/home policy (`CLAUDE_CONFIG_DIR`, `HERMES_HOME`, `home_policy = isolated` relocate or eliminate native homes). A skill is excluded from prompt sections A/B only when its winning root realpath is one of those resolved session-native roots. Native loading unproven (unknown provider, isolated home, env override elsewhere) ⇒ fail open to inclusion. Decisions tagged into observability/diagnostics. |
| **Expose manager** (`internal/skills`) | Create/remove/inspect per-skill symlinks in enabled preset roots; ownership authority = `skill_exposures` records, health reconciled from the filesystem (ADR-015); per-target preflight/commit/rollback state machines; safety rules (never overwrite, never copy, never touch foreign links). |
| **Settings section** (`internal/settings` + `internal/api`) | Two keys on the skills section; workspace scope opened for them; envelope gains daemon-computed `sources[]` + `inherits`. |
| **Command catalog** (`internal/daemon` session commands) | **Pre-overlay candidate projection** (B-005): candidates keyed by `RootID` + physical skill identity, computed before the name-keyed overlay. The effective winner projects the bare command; every eligible same-named candidate projects its qualified command (`<slug>:<name>` display, `RootID` identity), so shadowed definitions stay invocable per BR-12/US-008.EC-1. Expansion resolves the exact candidate by `RootID` and revalidates generation, source, and content. Collision diagnostics instead of silent drops. |
| **Web** (`web/src/systems/settings`, `systems/skill`, assistant-ui menu) | S1/S2/S3 per `_uiux.md`, bound to the daemon read models. |

Data flow: config (global ⊕ workspace overlay) → root resolution → scanner → registry (dedup, tiers, shadows, verification) → {prompt sections (filtered by injection policy), command catalog (unfiltered), settings/read APIs, watcher}. Expose writes filesystem links; inspection re-derives their health on read.

## Architectural Boundaries

No new packages. Changed packages and allowed imports (flow stays downward; `daemon/` remains the only multi-importer):

- `internal/config` — preset table + `SkillRootSpec` + resolution + validation. Imports nothing domain-side (unchanged posture). The table lives here because config already owns discovery-root vocabulary (`WorkspaceDiscoveryRoot`, `SkillsDirName`); `internal/skills` maps slugs to tiers.
- `internal/skillscan` — symlink-following + realpath reporting. No new imports.
- `internal/skills` — consumes `config.SkillRootSpec`; owns tier mapping, expose manager, per-root stats. Already imports `internal/config`.
- `internal/workspace` — scanner loop consumes resolved specs (already imports config + skillscan).
- `internal/settings` — section update/build for the two keys + workspace scope.
- `internal/daemon` — composition: watcher roots provider, harness `Provider` plumbing + injection policy, session-command source IDs, expose wiring into handlers.
- `internal/api/contract|core|httpapi|udsapi` — additive contract types, shared `BaseHandlers` methods, route registration on both transports (no transport-duplicated parsing).
- `internal/cli` — `skill sources`, `skill expose|unexpose`, `create --expose`, config-set kinds for the two keys.
- `internal/tools/builtin` — no shape change beyond additive `origin` in skill list/view payloads.

Forbidden: `internal/config` importing `internal/skills` (tier types stay out of config); any package other than `daemon/` wiring the injection policy into prompt assembly.

## Implementation Design

### Core Interfaces

```go
// internal/config/skill_sources.go
type SkillSourcePreset struct {
    Slug                   string // "agents", "claude" ("compozy" is the implicit builtin)
    Label                  string
    WorkspaceRel           string // ".agents/skills" — empty when the preset has no workspace root
    GlobalPath             string // "~/.agents/skills" — empty when global-less
    WorkspaceNativeReaders []string // canonical provider ids that natively read the WORKSPACE root
    GlobalNativeReaders    []string // canonical provider ids that natively read the GLOBAL root
    // Per-root granularity is load-bearing (B-003): openclaw+hermes read workspace
    // .agents/skills, but only openclaw reads ~/.agents/skills — Hermes's global home
    // is ~/.hermes/skills. claude reads both .claude roots.
    AlwaysOn  bool // true only for the implicit compozy entry
    DefaultOn bool // true for "agents"
}

func SkillSourcePresets() []SkillSourcePreset
func ValidateSkillSources(slugs []string) error // unknown slug → error with suggestion
// Batch allocator: groups all custom paths, assigns display slugs deterministically —
// sanitized basename; collisions get a short canonical-path-hash suffix. Pure function
// of the path SET (order-independent); never positional "-2" counters.
func CustomSourceSlugs(paths []string) map[string]string // canonical path → display slug

type RootOwnerScope string // "global" | "workspace" — ownership, NOT precedence
type RootKind string       // "builtin" | "preset" | "custom"

type SkillRootSpec struct {
    Dir         string         // absolute, expanded, canonicalized
    SourceSlug  string         // "compozy" | preset slug | custom display slug
    Kind        RootKind
    OwnerScope  RootOwnerScope // who owns the datum (workspace isolation boundary)
    WorkspaceID string         // canonical workspace id when OwnerScope == "workspace"
}

// RootID is the opaque, stable identity used for command-source identity,
// cache keys, exposure attribution, and drift checks. Derived ONLY from stable
// ownership + location fields: (OwnerScope, canonical WorkspaceID, Kind,
// canonical Dir) — NEVER from display slugs (mutable under collision
// re-allocation) or list position (B-004). Precedence tier is DERIVED in
// internal/skills (OwnerScope=workspace+preset/builtin → SourceWorkspace;
// global+preset/builtin → SourceUser; custom → SourceAdditional) and never
// stored on the spec.
func (s SkillRootSpec) RootID() string

// Replaces WorkspaceDiscoveryRoot.SkillsDir (deleted).
func (r WorkspaceDiscoveryRoot) SkillsDirs(cfg *SkillsConfig) []SkillRootSpec

// Global-side resolution for the registry and watcher.
func ResolveGlobalSkillRoots(cfg *SkillsConfig, home HomePaths) []SkillRootSpec
```

Custom-source display slugs: lowercase sanitized basename; collisions disambiguate with a
deterministic short content hash of the canonical path (`team-skills`, `team-skills-3f2a`) —
never list-position suffixes (identity must survive reordering).

```go
// internal/skills (types.go / registry)
type RegistryConfig struct {
    GlobalSkillRoots []compozyconfig.SkillRootSpec // replaces UserSkillsDir (deleted)
    GlobalAgentsDir  string                        // explicit — no more filepath.Dir derivation
    // ...existing fields unchanged
}

func sourceTierFor(spec compozyconfig.SkillRootSpec) SkillSource // workspace→SourceWorkspace, global→SourceUser, additional→SourceAdditional

// internal/skills/expose.go — ownership records persisted (ADR-015), health reconciled from fs.
// All four states are PUBLIC (B-006) — the surface renders and acts on each distinctly:
//   healthy          — link exists, Readlink == recorded LinkTarget, resolves into CanonicalDir
//   missing          — record exists, nothing at LinkPath (our link was deleted; re-expose repairs)
//   broken           — link exists AND Readlink == recorded LinkTarget, but resolution fails
//                      (canonical moved/deleted; provably OURS → unexpose/re-expose allowed)
//   foreign_conflict — entry exists at LinkPath but is not our link (Readlink != LinkTarget, or
//                      not a symlink) → report only; NEVER removed or overwritten
type ExposureStatus string // "healthy" | "missing" | "broken" | "foreign_conflict"

type ExposureRecord struct { // one row per created link — the ownership authority
    SkillName    string
    CanonicalDir string // realpath of the skill dir at creation
    Target       string // preset slug
    LinkPath     string
    LinkTarget   string // the literal symlink destination written at creation —
                        // the ownership proof that distinguishes broken (ours) from
                        // foreign_conflict (not ours) after the target stops resolving
    OwnerScope   compozyconfig.RootOwnerScope
    WorkspaceID  string // when OwnerScope == workspace
    CreatedAt    time.Time
    UpdatedAt    time.Time
}

type ExposureState struct {
    Record ExposureRecord
    Status ExposureStatus // reconciled: Lstat+EvalSymlinks vs Record, never fs-only inference
}

type TargetResult struct {
    Target   string
    Err      error          // deterministic code per _dx.md Errors
    Exposure *ExposureState // set on success
}

type ExposeManager struct{ /* store + config root resolution */ }

// Preflights EVERY target (name safety, enabled preset, containment, free path),
// then commits per target in order record→link; on mid-sequence failure rolls back
// completed targets and returns structured per-target results.
func (m *ExposeManager) Expose(ctx context.Context, skill *Skill, targets []string) ([]TargetResult, error)
func (m *ExposeManager) Unexpose(ctx context.Context, skill *Skill, targets []string) ([]TargetResult, error)
func (m *ExposeManager) Exposures(ctx context.Context, skill *Skill) ([]ExposureState, error)
// Reconcile record set vs filesystem for remove/update cleanup; retryable, idempotent.
func (m *ExposeManager) Reconcile(ctx context.Context, skill *Skill) ([]ExposureState, error)

// Destination safety: one canonical validator + resolver, applied before any write.
func validateExposeName(name string) error // single normalized segment, skill-name grammar,
                                           // rejects abs paths, separators, "..", NUL, encoded traversal
func resolveExposeDest(root, name string) (string, error) // realpathDeepestExisting parent +
                                                          // containment proof inside the preset root
```

**Expose lifecycle state machines** (B-005/B-007) — transition tables the implementation follows exactly:

| Operation | Order | Crash between steps reads as | Multi-target | Rollback |
|---|---|---|---|---|
| **Expose** (per target) | 1. preflight ALL targets (name, enabled preset, containment, free path) → 2. create absent preset-root dirs (exact root only, deepest-existing-parent containment, created dirs recorded in the operation) → 3. insert record → 4. create link | record without link = `missing` (repairable; never a foreign state) | sequential; mid-sequence failure triggers rollback of completed targets | remove created links, delete inserted records, remove **only** directories this operation created and that are empty; rollback failure → per-target cleanup errors + `skills.exposure.cleanup_failed` |
| **Unexpose** (per target) | 1. verify ownership (record + `LinkTarget` proof) → 2. remove link → 3. delete record | link gone, record present = `missing` → re-running unexpose deletes the record (idempotent). A record is never deleted before its link — a link without a record would read as foreign | per-target **independent** results (no all-or-nothing; removal is idempotent/retryable) | none needed — each target converges by re-run |
| **Skill remove / marketplace remove** | 1. resolve all records for the skill → 2. unexpose each (above) → 3. delete canonical dir **only after** every owned link is gone | canonical intact until cleanup completes | n/a | cleanup failure → abort removal with the retryable public error `skill_remove_blocked` (names the failing link); canonical dir preserved; records/links reflect exact progress |
| **Marketplace update** | path preserved in place; records untouched; reconcile after update verifies `healthy` | n/a | n/a | n/a |

Absent preset root (B-005): enabling a source never creates directories, and an absent root is a normal enabled state — expose is the one operation allowed to create the **exact preset root path** (workspace-level or global-level, per the skill's scope), with containment proven on the deepest existing parent; directories it created are part of the operation's rollback set (removed only if still empty).

```go
// internal/daemon (harness policy)
type HarnessSessionInput struct {
    // ...existing fields
    Provider string // NEW — populated from StartupPromptContext.Provider / session Info.Provider
}

type ResolvedHarnessPolicy struct {
    // ...existing fields
    SkillInjectionFilter func(skill *skillspkg.Skill) bool // false = suppress from prompt sections A/B
}

// Session-native-root resolution (B-003): computed once per session from the
// resolved provider + effective env/home policy; the filter suppresses a skill
// only when its winning root realpath ∈ this set. Empty set ⇒ nothing suppressed.
func resolveSessionNativeSkillRoots(provider ResolvedProvider, env providerenv.Effective) []string
```

```go
// internal/skillscan (scan result additions)
type RootScanStats struct {
    Truncated     bool
    SkippedLinks  []SkippedLink // {Path, Reason: dangling|escape|cycle}
}
```

### Data Models

**One SQLite change** (global stream): the `skill_exposures` side table. Every other datum stays schema-free. Storage decisions:

| Datum | Storage | Rationale (side-table-vs-JSON) |
|---|---|---|
| Enabled presets / custom roots | `config.toml` keys (global + workspace overlay) | Configuration, not runtime state; follows every existing skills key |
| Expose links | **Side table `skill_exposures`** (ownership authority) + filesystem reconcile (health) | A pure-filesystem model cannot distinguish "never exposed" from "our link was deleted", cannot prove a link is ours before removing it, and loses cleanup targets once the canonical skill disappears — US-011.AC-3/AC-5/EC-3..5 require ownership to outlive the link. Health (`healthy/missing/broken`) is still reconciled live from `Lstat`+`EvalSymlinks` against the record — the record is never trusted as current link state. Matchable state → side table, never JSON metadata (L-003 posture). |
| Per-root scan stats (counts, truncated, skipped links, collisions, verification summary) | In-memory registry projection, surfaced via envelopes | Ephemeral measurement, recomputed per scan |
| Suppression decisions | Not stored — observability tags + harness diagnostics | Decision log, not state |

`skill_exposures` columns (declarative schema + next gap-free Goose migration + `atlas.sum`/sqlc refresh via `make codegen`; fresh/reopen/ahead/integrity/equivalence suites extended — `eng-schema-migration`):

| Field | Type | Purpose |
|---|---|---|
| `id` | INTEGER PK | row identity |
| `skill_name` | TEXT NOT NULL | canonical skill name at creation |
| `canonical_dir` | TEXT NOT NULL | realpath of the skill dir the link must resolve into |
| `target_slug` | TEXT NOT NULL | preset slug the link was created in |
| `link_path` | TEXT NOT NULL UNIQUE | absolute link location |
| `link_target` | TEXT NOT NULL | literal symlink destination written at creation — ownership proof separating `broken` (ours) from `foreign_conflict` |
| `owner_scope` | TEXT NOT NULL | `global` \| `workspace` — workspace-isolation boundary |
| `workspace_id` | TEXT NULL | canonical workspace id when workspace-owned |
| `created_at` / `updated_at` | TIMESTAMP NOT NULL | lifecycle audit |

Integrity + query contract (N-001): `CHECK ((owner_scope = 'global' AND workspace_id IS NULL) OR (owner_scope = 'workspace' AND workspace_id IS NOT NULL))`; `UNIQUE (link_path)`; `UNIQUE (skill_name, owner_scope, workspace_id, target_slug)` (one link per skill×target×scope); index on `(skill_name)` (removal cleanup) and `(workspace_id)` (per-workspace exposure queries).

Config keys (rationale per field):

| Field | Type | Default | Purpose |
|---|---|---|---|
| `skills.sources` | `[]string` (preset slugs) | `["agents"]` | Which folder conventions are scanned besides compozy; validated against the preset table |
| `skills.custom_sources` | `[]string` (paths) | `[]` | Extra skills-only roots; `~`/absolute global, +workspace-relative at workspace scope |

Frontmatter allowlist extension (loader accepts-without-warning; **not honored**): `license`, `compatibility`, `allowed-tools`, `when_to_use`, `argument-hint`, `arguments`, `disable-model-invocation`, `user-invocable`, `disallowed-tools`, `model`, `effort`, `context`, `agent`, `background`, `hooks`, `paths`, `shell`, `aliases`.

Contract additions (additive; `make codegen` co-ship):

- Settings skills envelope: `config.sources`, `config.custom_sources`, read-model `sources[]{slug, label, kind: builtin|preset|custom, enabled, always_on, default, workspace_path?, global_path?, path?, roots[]}`, `inherits{sources, custom_sources}` at workspace scope.
- Per-root diagnostic schema (shared by settings envelope, `skill sources -o json`, and S1 — the single daemon-owned display source; every root object carries the full schema regardless of kind): `roots[]{root_id, path, exists, readable, scanned_count, skill_count, truncated, skipped_links[]{path, reason: dangling|escape|cycle}, collisions[]{name, winner_root_id, qualified_form}, verification{blocked, warned}, native_readers[]}`. `readable: false` (existing dir, permission denied) omits every count for the root — an unreadable root never renders as an empty one. `native_readers` is per root (B-003 granularity). Count semantics (N-001): `scanned_count` = SKILL.md candidates found in the root before dedup/verification; `skill_count` = effective winning catalog entries this root contributed. Runtime unavailable → counts omitted, never zeroed.
- Skill payloads (shared skill summary DTO consumed by `listSkills`, `getSkill`, `compozy__skill_list`, `compozy__skill_search`, `compozy__skill_view`, session command skill specs, and the extension Host API skill listing): `origin string` (source slug; empty for compozy). Detail payloads (`getSkill`, `compozy__skill_view`) additionally carry `exposures[]{target, path, status: healthy|missing|broken|foreign_conflict}`.
- `SkillRuntimeStatusPayload`: **unchanged** — dropped from scope (B-010/N-004); the settings envelope is the single sources read model.

### API Endpoints

Per `_dx.md` (shapes there are the contract):

- `GET /api/settings/skills` — extended envelope at both scopes (agent scope stays read-only for source keys).
- `PATCH /api/settings/skills` — **two scope-specific request shapes** (B-006):
  - Global scope: today's full-config body plus the two new list fields (unchanged model).
  - Workspace scope: a presence-aware override object — `{"override": {"sources": …, "custom_sources": …}}` where an **absent** field is untouched, **null** clears the override (returns to inherit), and an **array** sets it. Any other skills field present at workspace scope → 400 `workspace_scope_field_forbidden`. HTTP and UDS byte-identical. **Wire decoding (B-002)**: plain `*[]string` cannot distinguish absent from null in Go — the request DTO uses an explicit presence-aware wrapper (`type OptionalStringList struct { Present, Null bool; Value []string }` with custom `UnmarshalJSON`; absent → `Present=false`, `null` → `Present=true, Null=true`, array → `Present=true, Value=…`), parsed by ONE shared decoder used by both transports. This is the wire encoding of the tri-state merge (ADR-007). |
  - Validation errors: `unknown_skill_source`, `duplicate_skill_source`, `invalid_source_path`, `workspace_scope_field_forbidden` (400).
- `POST /api/skills/{name}/expose` · `POST /api/skills/{name}/unexpose` — body `{targets[], workspace_id?}` (`workspace_id` = canonical id only; CLI resolves ID/name/path before calling); response `{name, workspace_id?, results[]{target, ok, error?, exposure?}, rolled_back}` — `workspace_id` echoes the resolved canonical id on workspace-scoped operations and is **omitted** for global-skill operations; the same placement applies to the 409 failure envelope. Full success → 200; **any** failure (single- or multi-target) → 409 with the single failure envelope: top-level `expose_failed` + per-target results (`rolled_back: true` when compensation completed; per-target cleanup errors when it could not). Unexpose results are per-target independent (idempotent removal, no rollback). Per-target error codes: `skill_not_exposable`, `expose_target_disabled`, `expose_target_invalid`, `expose_name_conflict`, `expose_link_unsupported`, `expose_foreign_link`, `unsafe_skill_name`.
- `GET /api/skills/{name}` — `origin` + `exposures[]{target, path, status}`.
- Handlers land in `internal/api/core` `BaseHandlers`; HTTP and UDS register the same methods (no duplicated parsing). CLI `skill sources` reads the settings envelope (no new read route).

## Impact Analysis

| Component | Impact Type | Description and Risk | Required Action |
|---|---|---|---|
| `internal/config` | modified | New keys, table, resolution; low risk (additive + validated) | Follow `disabled_skills` template end-to-end |
| `internal/skillscan` | modified | Symlink following — medium risk (traversal surface) | Containment tests incl. macOS `/private/var` quirk |
| `internal/skills` registry | modified | Root lists + realpath dedup + stats — medium-high (three load paths + caches) | Guard `workspaceSkillPathsMatchRoots`; generation-keyed caches |
| `internal/daemon` harness/prompt | modified | Provider plumbing + filter — medium (hash short-circuit) | Filter before sha256; A+B together |
| `internal/settings` + API + CLI | modified | Workspace scope + expose endpoints — low-medium | Contract co-ship, parity tests |
| `web/` S1-S3 | modified | New section + chips + panel — low | Per `_uiux.md` component plan |
| Extensions/resource store | verified | Republication must carry new roots — high if missed (skills vanish after sync) | Explicit integration test |
| `internal/store` global stream | modified | New `skill_exposures` table — medium (append-only migration discipline) | Declarative schema + next Goose migration + `atlas.sum`/sqlc via `make codegen`; fresh/reopen/ahead/integrity/equivalence suites extended (`eng-schema-migration`) |

**Delete Targets** (same change, no fallback / no compat shim / no placeholder):

- `internal/skills/types.go` `RegistryConfig.UserSkillsDir` → replaced by `GlobalSkillRoots` + `GlobalAgentsDir`; every reader updated (`registry_load.go:25`, `registry_agent.go:387-395`, `watcher.go:60`, `internal/cli/skill_workspace.go:43`, `internal/daemon/boot_finalize.go:79-93`).
- `internal/config/agent.go` `WorkspaceDiscoveryRoot.SkillsDir()` → replaced by `SkillsDirs(cfg)`; callers `internal/workspace/scanner.go:162`, `internal/daemon/boot_finalize.go:142`.
- Hardcoded root joins in `internal/skills/registry_workspace_cache.go:271-293` → replaced by resolved specs.
- `internal/skills/registry_agent.go` `globalAgentsDir()` `filepath.Dir` derivation → explicit config field.
- Docs: `internal/CLAUDE.md` five-layer sentence amended to "each layer may own multiple roots"; `agents/definitions.mdx:324` + `agent-md.mdx:526` keep `skills.extra_sources` rejected, adding a pointer to runtime `skills.sources`.

## Extensibility Integration Plan

- **Extensions**: `resources.skills` manifest path unaffected; extension-published skills keep flowing through the resource store. The resource-authority handoff (`agent_skill_publisher.go` → `ApplyResourceRecords`) republishes from `DiscoverGlobal`/`DiscoverWorkspace`, which now carry configured roots — verified by integration test (risk table above). Extensions cannot register presets (ADR-014).
- **Hooks**: no new hook events; absorbed skills' `metadata.compozy.*` declarations (MCP servers, hooks, activation gates) follow the exact rules of user-tier skills today — same trust class as `~/.compozy/skills` content (all local user files), gated by the existing marketplace allowlists only for marketplace-tier skills.
- **Skills/capabilities**: skill catalog semantics unchanged; capability surface untouched (skills stay local artifacts per glossary).
- **Public skill-projection closure matrix** (B-010) — one row per surface, all co-shipped in the same change (OpenAPI/TS, native descriptors + schema digests, extension contract types, acpmock/matchers, CLI docs, official skill docs):

| Surface | `origin` | `exposures` | source diagnostics | Action |
|---|---|---|---|---|
| `GET /skills` (HTTP/UDS) + `compozy skill list` | yes | — | — | shared skill summary DTO gains `origin` |
| `GET /skills/{name}` + `skill info` | yes | yes | — | detail DTO gains both |
| `compozy__skill_list` / `compozy__skill_search` / `compozy__skill_view` | yes | view: yes | — | same shared DTOs; descriptor/schema digests refreshed |
| Session command skill specs (picker) | yes (label) | — | collision diagnostics | catalog spec gains origin label + qualified forms |
| Extension Host API `handleSkillsList` (`internal/extension/host_api_skills.go` + `extension/contract/skills.go`) | yes | — | — | consumes the shared summary DTO; contract type updated in the same change |
| Settings skills envelope + `skill sources` | — | — | full per-root schema | single sources read model |
| `SkillRuntimeStatusPayload` (`compozy status`) | no change | no change | no change | **explicit no-impact**: sources read model lives on the settings envelope only (N-004); payload untouched, verified by existing status tests |

- **Registries/bridge SDKs/MCP sidecars/protocol docs**: unaffected — checked surfaces: bridge SDK types (no skill payloads), MCP sidecar lifecycle (per-skill decls unchanged), network protocol (skills never cross the wire).

## Agent Manageability Plan

Consistent with `_dx.md` (authoritative shapes there): inspect via `compozy skill sources -o json` / `GET /api/settings/skills` (both scopes); configure via `compozy config set skills.sources|custom_sources` (global/workspace scope; daemon-routed) and `PATCH /api/settings/skills`; operate expose via `compozy skill expose|unexpose` / expose endpoints; diagnose via `skill where`, `skill info`, per-root `readable`/`truncated`/skips in the sources read model, and suppression reasons in harness diagnostics. **Authorization split (grill 2026-08-20):** expose/unexpose are skill operations — allowed to agent-scope callers under the same gate as skill enable/disable; the two source config keys remain global/workspace policy, read-only at agent scope. UDS carries parity for every route. Deterministic error codes enumerated in `_dx.md` Errors. `tool_surface` registers both keys as agent-writable string slices at global+workspace scope (trust-root gate: same class as `disabled_skills`).

## Config Lifecycle

- **Keys/defaults**: `skills.sources = ["agents"]`, `skills.custom_sources = []` in `SkillsConfig` (`config_extensions_sandbox.go`) + `defaults.go`.
- **Merge/overlay**: pointer-typed `*[]string` in `skillsOverlay` with replace-on-present semantics (`merge.go`, `merge_features.go`, `merge_overlay.go`); workspace overlay carries the same fields.
- **Validation**: `(SkillsConfig).Validate()` — slug membership (did-you-mean), path shape per scope, cross-key duplicate-root rejection.
- **Lifecycle matrix**: both keys `Live` (`lifecycle.go` rules before the `skills.*` RestartRequired catch-all); site `lifecycle-matrix.mdx` rows.
- **Surfaces**: `tool_surface.go` registration; CLI set-kind `configSetStringSlice`; settings apply path (`section_feature_apply.go`, `section_skills_update.go` + workspace scope with the presence-aware override request shape — the `OptionalStringList{Present, Null, Value}` wrapper and the one shared HTTP/UDS decoder per the API Endpoints section; plain `*[]string` stays only in the TOML overlay model, where JSON-null semantics don't exist).
- **Docs**: `config-toml.mdx` (section index, example block, field table), `file-locations.mdx` (location tables + tree), `skills/index.mdx` (discovery order + precedence), generated CLI docs via `make codegen`; official skill `skills/compozy/references/configuration.md` + `references/tools-and-skills.md`.
- **Tests**: defaults, merge/overlay tri-state, validation errors, lifecycle classification, CLI parse (comma + JSON forms) — enumerated in `_tests.md`.

## Testing Approach

- **Unit** (per package, `-race`): config table/resolution/validation; scanner symlink matrix (follow/dedup/escape/dangling/cycle/caps) on temp dirs; registry root-list loading + realpath dedup + tier mapping; expose manager fs semantics; harness filter decisions; settings section scope rules.
- **Integration** (`+integration`): live apply end-to-end (config change → watcher → registry → counts), workspace override isolation (two workspaces, disjoint roots), resource republication after source change, marketplace remove/update × expose links.
- **E2E runtime** (Go harness + acpmock): session picker catalog includes absorbed skills with origins; invocation of an absorbed skill; suppression per provider (mock provider identities); drift rejection when a source is disabled mid-flight. Contract changes co-ship mock/matcher updates (runtime-contract co-ship rule).
- **E2E web** (Playwright): S1 toggle→count flow, custom add/remove, workspace override inherit/override states, expose panel actions, picker chip visibility.
- Fixture strategy: real temp filesystems for every scan/expose test (no fs mocks); acpmock for provider identity; no network.
- Test placement (N-003): every `_tests.md` group names its owning canonical suite; `cy-create-tasks` carries those annotations into task bodies, and a standalone regression is created only where no canonical suite can own the invariant (`eng-consolidate-test-suites`).
- Observability: one coverage-matrix suite owns the canonical-event contract (Monitoring section) and fails when a lifecycle path omits its event.

## Development Sequencing

### Build Order

1. **Config foundation** — table, keys, resolution, validation, overlay, lifecycle, tool_surface, CLI parsing. Gate: unit.
2. **Discovery core** — `SkillsDirs`/global resolution consumers (3 scan paths), scanner symlink+dedup+stats, `workspaceSkillPathsMatchRoots`, generation-keyed caches, watcher roots provider, live apply + republication. Gate: unit + integration.
3. **Contracts/APIs** — `skill_exposures` schema + migration first (codegen gate), then settings envelope + workspace-override PATCH shape, expose endpoints + `ExposeManager` (records + reconcile + rollback), CLI verbs (`sources`, `expose|unexpose`, `create --expose`), origin fields across the closure matrix, `make codegen`. Gate: unit + `codegen-check`.
4. **Session layer** — harness `Provider`, injection filter (A+B, pre-hash), command-catalog source IDs + collision diagnostics. Gate: unit + e2e-runtime.
5. **Web** — S1/S2/S3 per `_uiux.md`. Gate: web lanes via turbo.
6. **Docs + official skill + QA tail** — site pages, `skills/compozy/` references, `internal/CLAUDE.md` amendment, QA scenarios per tracker rules. Gate: `make gate-full` at workstream close.

Phases 1-2 are behavior-additive but delete the single-root fields (hard cut inside phase 2, one commit batch). No cleanup-only phase needed — delete targets fall inside phase 2/3 edits.

### Technical Dependencies

None external. Internal blocker order: phase 2 needs 1; 3-4 need 2; 5 needs 3.

## Monitoring and Observability

**Canonical durable observe events** (B-011) — appended to the operational ledger with correlation keys `{workspace_id?, config_generation, actor_kind, actor_id}` plus the fields below; a single coverage-matrix suite fails when any lifecycle path omits its event (per `internal/CLAUDE.md` Observability). Hooks, where any fire, dispatch at these call sites — never by tailing the event table.

All events carry the base correlation keys `{workspace_id?, config_generation, actor_kind, actor_id}`. Durable append happens **before** any live revision/SSE broadcast derived from the same change (B-011).

| Event | Extra fields | Emitted at |
|---|---|---|
| `skills.sources.applied` | scope, generation, root count per source | apply coordinator successful commit only |
| `skills.sources.superseded` | discarded_generation, winning_generation | apply coordinator discarding stale work — never recorded as applied |
| `skills.sources.apply_failed` | scope, generation, error class | apply validation/commit failure |
| `skills.scan.truncated` | root_id, path, scanned, cap | scanner pass |
| `skills.scan.link_skipped` | root_id, path, reason | scanner pass |
| `skills.exposure.created` / `skills.exposure.removed` | skill, target, link_path | expose/unexpose commit per target |
| `skills.exposure.operation_failed` | skill, target, link_path?, error class | preflight/commit failure per target (validation, conflict, link-unsupported) |
| `skills.exposure.broken_detected` | skill, target, link_path, status | reconcile (incl. `foreign_conflict` detection) |
| `skills.exposure.cleanup_failed` | skill, target, link_path, error class | rollback/reconcile failure (partial-failure path) |

**Log-only** (slog, not durable events): per-suppression decisions `skills.injection.suppressed` {session_id, skill, source slug, provider} — high-cadence, session-scoped; carried in harness observability tags (`buildHarnessObservabilityTags`) and harness diagnostics ("why isn't this skill injected").

Settings envelope is the operator-facing meter (labels, counts, exists, truncated, skips, collisions per root).

## Technical Considerations

### Key Decisions

Decisions not already carrying an ADR: preset table lives in `internal/config` (avoids a new leaf package; config already owns discovery-root vocabulary; skills maps slugs→tiers to keep the tier enum out of config) · `skill sources` CLI reads the settings envelope instead of a new route (one read model, no drift) · expose handlers live in `internal/api/core` `BaseHandlers` (transport parity by construction).

### Known Risks

- **Resource-authority staleness** (highest): configured sources visible pre-sync, gone post-sync if republication misses them — dedicated integration test; risk table row.
- **Cache generation bugs**: stale workspace projections after live apply — generation-keyed invalidation + the latest-generation commit fence (Safety Invariant 9) + deterministic race test (IT-012).
- **Hash short-circuit**: filtering after the augmenter's sha256 would freeze suppression — filter placement test asserts pre-hash.
- **Provider id drift**: canonical vocabulary frozen in Assumptions (`claude`, `openclaw`, `hermes`); typed classifier resolved with the agent — covered by UT-085 and IT-008.
- **Exposure record/filesystem divergence**: records are ownership authority, filesystem is health truth — reconcile is retryable/idempotent (IT-013); partial-failure paths emit `skills.exposure.cleanup_failed`.

## Safety Invariants

1. **Per-projection containment** (B-004): a followed symlink loads only when its resolved target lies inside that projection's trusted set — the owning workspace's configured roots plus the intentionally visible global roots, **never another workspace's roots**. A symlink from workspace A resolving into workspace B's configured root is rejected. Escaping, dangling, and cyclic links are skipped with a per-entry diagnostic — never followed, never fatal to the root.
2. One physical skill directory (realpath identity) yields exactly one catalog entry per scope, attributed to the highest-precedence root that reached it; CompozyOS's own expose links can never shadow their canonical skill.
3. **Expose per-target discipline**: every target is preflighted (name safety, enabled preset, containment, free path) before any mutation; operations follow the lifecycle state machines exactly — expose commits record→link, unexpose removes link→record, skill removal deletes the canonical dir only after every owned link is cleaned; mid-sequence expose failure rolls back completed targets and returns structured per-target results; rollback/cleanup failure surfaces per-target errors — no silent residue. Allowed filesystem mutations per operation: create the exact absent preset root (recorded), create a link at a free path, remove a link proven ours, remove empty directories this operation created; never overwrite, never copy, never touch a foreign entry.
4. **Ownership authority**: a link is "ours" if and only if a `skill_exposures` record matches its path AND `Readlink(link) == record.LinkTarget`; filesystem shape alone is never interpreted as ownership. Record without a live link → `missing`; ours-but-unresolvable → `broken` (repair allowed); entry at the path that is not our link → `foreign_conflict` (report only, never mutated).
5. **Expose destination path-safety**: the target name passes `validateExposeName` (single normalized segment, skill-name grammar; rejects absolute paths, separators, `..`, NUL, encoded traversal, Unicode tricks) and `resolveExposeDest` proves the deepest-existing parent stays inside the preset root — before any write, including preset-root creation.
6. Injection suppression filters prompt sections A and B only; the command-catalog projection, settings/read APIs, enable/disable, shadows, and explicit `/<skill>` expansion are never filtered; unknown session provider ⇒ no suppression.
7. **Opaque stable source identity**: a root's identity is `RootID` — derived from exactly (owner scope, canonical workspace id, root kind, canonical dir), byte-identical to the Core Interfaces and ADR-016 definition; display slugs and list position never participate. It is deterministic for a given config generation across catalog build and invocation expansion; a generation change invalidates in-flight invocations through the existing source-drift rejection — stale content is never served.
8. Config writes per scope remain sequential (existing discipline); root resolution is atomic per generation — no scan pass mixes roots from two generations.
9. **Latest-generation commit fence** (B-009): one apply coordinator owns the monotonic generation; every asynchronous commit point (registry swap, resource-authority publication, diagnostics replacement, session-command revision broadcast) re-compares the current generation immediately before committing and discards or cancels stale work — generation N can never overwrite N+1 on any read surface.
10. `VerifyContent` runs on every non-bundled skill on every load regardless of source; critical findings block that skill only, never the root.
11. Per-root scan caps hold (300 candidates / 20k entries); exceeding marks `truncated` in the same pass and surfaces in every sources read model.
12. Workspace skill-cache keys incorporate the owning workspace id and the effective source-config generation; two workspaces, or one workspace before/after an override change, never share projections.

## File References

### Repo Files

- `internal/config/agent.go:155-202` — `WorkspaceDiscoveryRoots` + `SkillsDir()`; the resolution seam being generalized.
- `internal/config/home.go:18-20,169-250,317-355` — dir-name constants, home resolution, `expandUserPath` (reuse, don't duplicate).
- `internal/config/config_extensions_sandbox.go:9-17` — `SkillsConfig` (field site); `internal/config/defaults.go:75-78` — defaults.
- `internal/config/merge.go:346-353` + `merge_features.go:12-29` + `merge_overlay.go:26` — overlay template (`disabled_skills`).
- `internal/config/lifecycle/lifecycle.go:82,145,180-182` — Live vs RestartRequired rules ordering.
- `internal/config/tool_surface.go:154-156` + `tool_surface_security.go:102` — agent-writable key registration + trust gate.
- `internal/cli/config_value_parse.go:173-192` + `internal/cli/config.go:271-274` — list-value parsing + set-kind table.
- `internal/cli/config_value_commands.go:35-77,205-213` — `--scope`/`--workspace` flags on set/unset (surface reused by dx examples).
- `internal/cli/skill_commands.go:21-107` + `skill_commands_mutation.go:20-65,102-103` — verb family, `where`, `create` target.
- `internal/cli/skill_workspace.go:43` + `internal/cli/config_daemon_mutation.go:72-86` — local fallback registry + daemon-routed skills.* writes.
- `internal/skillscan/scan_directory.go:18-33,57-63,112-116` + `scan.go:17-53` — walker, caps, dot-dir skip, symlink posture being changed.
- `internal/skills/types.go:96-111,227-233` — tier enum + `RegistryConfig` (delete target field).
- `internal/skills/registry_load.go:14-38,157-218` — global load loop + verification + marketplace provenance flip.
- `internal/skills/registry_workspace_cache.go:225-293,295-339,382-416` — workspace roots, backward overlay order, consistency guard, cache keys.
- `internal/skills/registry_agent.go:101-121,387-395` — agent-local overlay + `globalAgentsDir` derivation (delete target).
- `internal/skills/registry_override_log.go:52-62` + `registry_snapshot.go:51-96` + `shadows.go` — overlay/shadow machinery absorbed unchanged.
- `internal/skills/watcher.go:26,55-100,138-173,235-273,329-359` — polling, roots provider, snapshot branching on `agents` basename.
- `internal/skills/registry_discovery.go:76-137,155-214` + `internal/daemon/agent_skill_publisher.go:138-230` — resource-authority handoff (highest-risk seam).
- `internal/skills/catalog.go:17-21,57-113,128-187` — prompt catalog build + unchanged-marker (injection A).
- `internal/daemon/prompt_skills.go:74-160` — per-turn augmenter + sha256 short-circuit (injection B; filter goes before hash).
- `internal/daemon/harness_context.go:116-128,168-177,199-235,267` + `harness_context_session.go:9-18` + `harness_context_policy.go:3-53` — harness resolver gaining `Provider` + filter.
- `internal/daemon/session_commands.go:69-106,108-152,192-223,241-250` — expansion, projection, skill specs, `sameSkillSource` drift check.
- `internal/command/catalog.go:20-107,145-181` + `internal/command/instructions.go:12-60` — token precedence, qualified descriptors, budgets.
- `internal/session/command_parser.go:10-31` + `manager_prompt_submit.go:103,172,293-309` — invocation pipeline (order: expand → augment).
- `internal/settings/section_skills_update.go:17-131` + `section_feature_apply.go:15-44` + `section_build.go:39-90` + `models_runtime.go:39-47` — section scope rules, TOML apply, envelope build.
- `internal/api/core/skills.go` + `status_skills.go:14-44` + `conversions_settings_sections.go:82-93,461-464` + `settings_section_requests.go:205-225` + `internal/api/contract/settings_mutations.go` + `contract/status.go:104-110` — handler/contract seams.
- `internal/api/httpapi/routes.go:258-267,396-397` + `internal/api/udsapi/routes.go:285-295,409-410` — route registration parity.
- `internal/tools/builtin/skills.go:13-56` — native tools whose payloads gain `origin`.
- `web/src/routes/_app/settings/-skills-settings-page.tsx` + `web/src/systems/settings/hooks/use-settings-skills-page.ts:122-213` + `hooks/settings-skills-draft-logic.ts` — page wiring + draft kinds (add `sources` kind).
- `web/src/systems/settings/components/settings-disabled-skills-section.tsx` + `settings-taglist-field.tsx` + `components/index.ts` — component patterns to copy (S1).
- `web/src/systems/settings/adapters/settings-sections-api.ts:165-199` + `hooks/use-settings-mutations.ts:193-207` + `lib/sections.ts:75-80` — adapter/mutation/keywords.
- `web/src/systems/session/hooks/use-session-commands.ts:92-129` + `lib/session-command-menu-model.ts:242-249` + `web/src/components/assistant-ui/session-composer-command-menu.tsx:34-240` — picker (S2).
- `web/src/routes/_app/marketplace.skills.tsx` + `web/src/systems/marketplace/components/marketplace-detail-skill-installed.tsx` + `web/src/systems/skill/**` — catalog/detail (S3).
- `skills/compozy/references/tools-and-skills.md` + `references/configuration.md` — official-skill updates.
- `packages/site/content/docs/configuration/{config-toml,file-locations,lifecycle-matrix,skill-md}.mdx` + `content/docs/skills/index.mdx` + `agents/definitions.mdx:324` + `configuration/agent-md.mdx:526` — docs surface.

### Competitor References

- `.resources/claude-code/skills/loadSkillsDir.ts:67,216,270,340` — native skill loading, `userInvocable` default, `skillRoot` internals that never cross the SDK boundary (grounds ADR-009).
- `.resources/claude-code/commands.ts:728` — description-string origin decoration (why it is not a dedup contract).
- `.resources/openclaw/src/acp/commands.ts:1-50` — hardcoded ACP built-ins (no skill announcement).
- `.resources/hermes/acp_adapter/server.py:581-621,2068-2109` — hardcoded advertised commands.
- vercel-labs/skills (`src/agents.ts`, `src/installer.ts`, `src/constants.ts`) — **not vendored under `.resources/`**; cited by URL in `analysis/ecosystem-patterns.md` (preset taxonomy + symlink install strategy + claude-code exemption).

### Design and Analysis Sources

- `analysis/backend-runtime.md` — discovery/registry/watcher map; feeds ADR-001/003/005/007 and every §Implementation seam.
- `analysis/web-settings.md` — settings envelope/flow/component inventory; feeds `_uiux.md` S1 and ADR-006.
- `analysis/ecosystem-patterns.md` — folder-pattern taxonomy + spec/superset frontmatter; feeds ADR-002/008/011/012.
- `analysis/acp-commands.md` — protocol verdict; feeds ADR-009/010.
- `analysis/slash-picker-injection.md` — picker/injection pipeline + gaps G1-G6; feeds ADR-009/013 and the harness design.

## Assumptions and Defaults

- Defaults: `sources = ["agents"]`, `custom_sources = []`; existing `poll_interval` (3s) governs live pickup; scan caps stay 300/20k per root. Live-apply visibility budget: a saved source change reaches every read surface within two poll intervals (owned by IT-001).
- Provider ids are the **canonical provider registry ids**: `claude`, `openclaw`, `hermes` (`claude-code → claude`, `hermes-agent → hermes`). Native-reader knowledge is **per root**, not per preset: workspace `.agents/skills` → `openclaw`+`hermes`; global `~/.agents/skills` → `openclaw` only (Hermes's global home is `~/.hermes/skills`); both claude roots → `claude`. Suppression uses the session's **resolved** native roots (provider × per-root knowledge × effective env/home policy — `CLAUDE_CONFIG_DIR`, `HERMES_HOME`, `OPENCLAW_STATE_DIR`, `home_policy = isolated` relocate or eliminate them) and matches winning-root realpaths; unproven native loading fails open to inclusion. The public `sources[].roots[].native_readers` field carries the same per-root canonical vocabulary.
- Custom-source display slug: lowercase basename sanitized to `[a-z0-9-]`; collisions disambiguate with a deterministic short hash of the canonical path (stable under list reordering). Identity everywhere else is `RootID`, never the display slug.
- Preset roots that don't exist are valid (absent state); no directory is ever created by enabling a source.
- Expose links are relative-path symlinks when target and canonical share a filesystem root scope (workspace), absolute otherwise.
- The `compozy` implicit source contributes the existing roots exactly as today (including agent-local overlays and marketplace provenance); nothing about tier semantics changes for it.
- Windows: symlink creation failure surfaces `expose_link_unsupported`; scanning follows links only where the OS resolves them (no junction special-casing in v1).
- Zero enabled optional sources + empty custom = compozy-only discovery; a valid, explicitly rendered state.

## Architecture Decision Records

- [ADR-001: Two separate config keys — preset slugs plus free-form custom roots](adrs/adr-001.md)
- [ADR-002: v1 preset curation — `agents` on by default, `claude` available](adrs/adr-002.md)
- [ADR-003: The `compozy` preset is always on and implicit](adrs/adr-003.md)
- [ADR-004: `custom_sources` ships in v1](adrs/adr-004.md)
- [ADR-005: Source configuration applies live](adrs/adr-005.md)
- [ADR-006: Workspace-scope override in v1 with full CLI + API + web management](adrs/adr-006.md)
- [ADR-007: Tri-state replace merge; six tiers keep root lists](adrs/adr-007.md)
- [ADR-008: Ecosystem frontmatter recognized without warnings, not honored](adrs/adr-008.md)
- [ADR-009: Prompt-injection-only suppression via provider→native-roots mapping](adrs/adr-009.md)
- [ADR-010: Suppression is automatic — no config key](adrs/adr-010.md)
- [ADR-011: Canonical-home create + symlink expose (verb, create option, web; preset targets only)](adrs/adr-011.md)
- [ADR-012: First-level symlink following with realpath dedup](adrs/adr-012.md)
- [ADR-013: Skill origin visible in picker, settings, CLI](adrs/adr-013.md)
- [ADR-014: v1 non-goals — no remote installer/sync, skills only, closed preset table, symlinks never copies](adrs/adr-014.md)
- [ADR-015: Exposure ownership records persisted in a side table; health reconciled from the filesystem](adrs/adr-015.md)
- [ADR-016: Pre-overlay candidate projection with opaque stable root identity](adrs/adr-016.md)
