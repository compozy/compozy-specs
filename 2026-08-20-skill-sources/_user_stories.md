# User Stories: Skill Sources

Canonical behavior catalog for configurable skill source folder patterns. Companion to `_spec.md`; consumed by `_spec.md` Part II (component mapping), `_uiux.md` (surface states), and `_tests.md` (coverage matrix).

## Personas

- **Operator** — runs CompozyOS on their machine alongside other agent tools (Claude Code, Codex, Cursor, …); already keeps skills in ecosystem folders (`~/.claude/skills`, `~/.agents/skills`) and wants CompozyOS to see them without moving or duplicating anything.
- **Team member** — works in a repository that standardizes a committed skill folder (`.agents/skills`); expects every teammate's CompozyOS to pick those skills up with zero per-machine setup.
- **Skill author** — creates and maintains skills from inside CompozyOS and wants selected skills readable by external tools through their native conventions.
- **Managing agent** — a CompozyOS session (or external automation) that inspects, configures, and repairs the skill-source setup through structured CLI/HTTP/UDS surfaces, never the web UI.

## Story Index

| ID     | Feature Area         | Persona        | Story                                                        |
| ------ | -------------------- | -------------- | ------------------------------------------------------------ |
| US-001 | Source configuration | Operator       | Universal folders load by default                             |
| US-002 | Source configuration | Operator       | Enable/disable a source preset and see it apply live          |
| US-003 | Source configuration | Operator       | Add custom skill directories                                  |
| US-004 | Workspace behavior   | Team member    | Committed repo skills load for every teammate                 |
| US-005 | Workspace behavior   | Team member    | Override enabled sources for one workspace                    |
| US-006 | Discovery semantics  | Operator       | Same-named skills across sources resolve predictably          |
| US-007 | Discovery semantics  | Operator       | Externally installed (symlinked) skills appear once           |
| US-008 | Session usage        | Operator       | Slash picker lists skills from every source with their origin |
| US-009 | Session usage        | Operator       | No duplicated skill context when the provider loads natively  |
| US-010 | Expose               | Skill author   | Create a skill and expose it to other tools in one step       |
| US-011 | Expose               | Skill author   | Expose/unexpose existing skills and trust cleanup             |
| US-012 | Agent management     | Managing agent | Inspect and change source config through structured surfaces  |
| US-013 | Settings UI          | Operator       | Manage sources from Settings > Skills                         |
| US-014 | Diagnostics          | Operator       | Truncation and name collisions are visible, never silent      |
| US-015 | Discovery semantics  | Operator       | Ecosystem skill definitions load without warning noise        |

## Source configuration

### US-001: Universal folders load by default

**As an** operator, **I want** skills in `~/.agents/skills` and `<workspace>/.agents/skills` loaded by default, **so that** my existing cross-tool skills work in CompozyOS without setup.

Acceptance criteria:

- AC-1: Given a fresh CompozyOS install and a populated `~/.agents/skills`, when the daemon starts, then those skills appear in the skill catalog with their origin identified.
- AC-2: Given a workspace containing `.agents/skills/<name>/SKILL.md`, when the workspace is used, then `<name>` is available in sessions of that workspace.
- AC-3: Given the default configuration, when the operator inspects active sources, then the universal folders and the Compozy folders are listed as enabled, with per-source skill counts.

Edge cases:

- EC-1: `~/.agents/skills` does not exist → the source shows as enabled with zero skills and a "directory absent" state; no error, no log spam.
- EC-2: A skill directory without `SKILL.md` inside the root → ignored, not an error.
- EC-3: A `SKILL.md` without the required name field → skill rejected with a per-skill diagnostic; other skills in the root load normally.
- EC-4: First run on a machine with hundreds of skills across folders → catalog loads them up to documented per-root limits; anything beyond is reported as truncated (US-014).

### US-002: Enable/disable a source preset and see it apply live

**As an** operator, **I want** to toggle a known source pattern (e.g. the Claude folders) on or off, **so that** I control exactly which conventions CompozyOS absorbs.

Acceptance criteria:

- AC-1: Given the claude preset is off, when the operator enables it, then skills in `.claude/skills` and `~/.claude/skills` appear in the catalog within seconds, without a daemon restart.
- AC-2: Given the preset is on, when the operator disables it, then its skills leave the catalog and the session picker on the next refresh.
- AC-3: The compozy source is always listed as active and cannot be disabled.
- AC-4: Given a save, when it completes, then the interface states whether it applied immediately, matching what the daemon reports.

Edge cases:

- EC-1: Disabling a source whose skill is mid-conversation → already-delivered skill content in that conversation is unaffected; the skill stops being offered from the next turn on.
- EC-2: Rapid repeated toggling → last saved state wins; no residual skills from intermediate states.
- EC-3: Unknown preset name submitted through any surface → deterministic validation error naming valid slugs and the closest match; nothing is applied.
- EC-4: Empty preset list → valid state meaning "Compozy folders only"; the settings surface shows an explicit "defaults only" state, not an error.

### US-003: Add custom skill directories

**As an** operator, **I want** to register arbitrary directories as skill sources, **so that** team- or company-specific skill layouts work without waiting for a preset.

Acceptance criteria:

- AC-1: Given the operator adds `~/team-skills`, when it is saved, then skills in that directory appear in the catalog, attributed to that custom source.
- AC-2: Custom entries accept absolute paths and home-relative (`~/…`) paths at global scope, plus workspace-relative paths at workspace scope.
- AC-3: Removing a custom entry removes its skills from the catalog on the next refresh.
- AC-4: A custom source's display label derives from its directory basename; when two custom paths share a basename, the collision is disambiguated with a deterministic short suffix — visible wherever origin labels render (lists, picker, qualified invocation forms).

Edge cases:

- EC-1: Path does not exist → accepted, listed with a "directory absent" state and zero count; skills appear automatically if the directory is created later.
- EC-2: Duplicate of an already-active root (same resolved path, via preset or custom) → rejected with a message naming the existing source.
- EC-3: Workspace-relative path submitted at global scope → rejected with a scope-specific message.
- EC-4: A custom root that resolves (via symlink) outside any trusted location → the escaping entry is skipped with a diagnostic; it never loads.
- EC-5: A custom root pointed at an enormous tree (e.g. `~`) → scan stops at documented limits and reports truncation (US-014).
- EC-6: Directory exists but cannot be read (permission denied) → the source shows an explicit unreadable state with no counts (never a zero that looks measured); other roots load normally; fixing permissions resolves it on the next refresh.

## Workspace behavior

### US-004: Committed repo skills load for every teammate

**As a** team member, **I want** skills committed under the repo's `.agents/skills` to load for everyone who opens the workspace, **so that** the team shares one skill set through version control.

Acceptance criteria:

- AC-1: Given a repo with `.agents/skills/review-checklist/SKILL.md`, when any operator with default config uses that workspace, then `review-checklist` is available in sessions there.
- AC-2: The skill is workspace-scoped: other workspaces do not see it.

Edge cases:

- EC-1: The same skill name also exists globally → the workspace copy wins for that workspace; the global one is recorded as shadowed (US-006).
- EC-2: Checking out a branch that deletes the skill directory → the skill leaves the catalog on the next refresh; sessions no longer offer it.

### US-005: Override enabled sources for one workspace

**As a** team member, **I want** one workspace to enable or restrict sources independently of my global setup, **so that** repo-specific policy does not leak across projects.

Acceptance criteria:

- AC-1: Given global sources `[agents]`, when the workspace override sets `[agents, claude]`, then only that workspace absorbs the Claude folders; other workspaces still use the global list.
- AC-2: A workspace with no override inherits the global configuration, and the management surfaces state that it is inheriting.
- AC-3: The override is manageable from the settings page, the CLI, and the API at workspace scope (operator and managing-agent parity).
- AC-4: Removing the override returns the workspace to inherited behavior.

Edge cases:

- EC-1: Override set to an empty list → that workspace scans Compozy folders only, even though global enables more.
- EC-2: Conflicting concurrent edits at global and workspace scope → both persist independently; effective behavior recomputes from the merged result; the settings surface shows the effective outcome.
- EC-3: Custom sources overridden while presets are inherited (only one key present) → each key merges independently; the surface shows which of the two is overridden.

## Discovery semantics

### US-006: Same-named skills across sources resolve predictably

**As an** operator, **I want** deterministic resolution when two sources carry the same skill name, **so that** I always know which version a session gets.

Acceptance criteria:

- AC-1: Given `deploy-check` in both the workspace Compozy folder and the workspace universal folder, when the catalog resolves, then the Compozy copy wins and the other is recorded as shadowed.
- AC-2: Given `deploy-check` in a workspace source and a global source, the workspace copy wins.
- AC-3: The losing definitions are inspectable: the operator can see every path that participated and which one won.

Edge cases:

- EC-1: Shadowing appears/disappears as sources toggle → resolution recomputes on refresh; the shadow record always reflects the current winner.
- EC-2: Two same-named skills inside one root (nested group dirs) → last-scanned wins per existing rule; recorded as a shadow, not silently dropped.
- EC-3: Name collision between a skill and an existing session command (builtin or agent-announced) → the skill remains reachable through its qualified form; the collision is visible in diagnostics (US-014).

### US-007: Externally installed (symlinked) skills appear once

**As an** operator, **I want** skills installed by external tools as symlinks (canonical body + per-tool links) to appear exactly once, **so that** popular installer layouts don't produce duplicates or gaps.

Acceptance criteria:

- AC-1: Given `~/.claude/skills/foo` → `~/.agents/skills/foo` (installer-created symlink) and both presets enabled, when the catalog resolves, then `foo` appears once.
- AC-2: Given only the claude preset is enabled, the symlinked `foo` still loads (first-level links are followed).

Edge cases:

- EC-1: Symlink whose target was deleted (dangling) → skipped with a per-entry diagnostic; not an error for the whole root.
- EC-2: Symlink resolving outside every trusted skill location → skipped with a diagnostic; never followed.
- EC-3: Symlink cycle at first level → detected and skipped; scan completes.
- EC-4: CompozyOS's own expose links (US-010/011) inside scanned roots → never produce a duplicate or self-shadow.
- EC-5: A link inside one workspace's folder resolving into another workspace's configured folder → rejected with a diagnostic; workspaces never read each other's roots through links.

## Session usage

### US-008: Slash picker lists skills from every source with their origin

**As an** operator, **I want** typing `/` in the session composer to list skills from all enabled sources, **so that** I can invoke any of them exactly like Compozy-native skills.

Acceptance criteria:

- AC-1: Given enabled sources with skills, when the operator types `/` in the composer, then those skills appear in the picker's skills section alongside Compozy-native ones.
- AC-2: Skills from non-Compozy sources carry a discreet origin label; Compozy-native skills stay unlabeled.
- AC-3: Selecting a listed skill and submitting delivers the skill's content into the conversation turn, with the same verification and budgets as native skills.
- AC-4: Enabling a source mid-session makes its skills appear in the picker without recreating the session.

Edge cases:

- EC-1: Two enabled sources carry the same bare name → both remain invocable through qualified forms; the picker distinguishes them by origin label.
- EC-2: Skill fails content verification at invocation time → the turn is rejected with the existing deterministic error; nothing partial is injected.
- EC-3: Source disabled between picking and submitting → invocation fails with the existing "source drift" rejection, not a silent fallback.
- EC-4: Skill body over the per-skill budget → truncated per existing invocation budget rules; behavior identical to native skills.

### US-009: No duplicated skill context when the provider loads natively

**As an** operator, **I want** CompozyOS to avoid re-sending skills my session's provider already loaded from its own folders, **so that** context stays lean without losing manageability.

Acceptance criteria:

- AC-1: Given a Claude Code session and the claude preset enabled, when the session's prompt context is assembled, then skills whose winning root is a Claude-native folder are omitted from the injected catalog.
- AC-2: Those same skills remain visible in the picker, the skills lists, and enable/disable — only prompt injection is suppressed.
- AC-3: Explicitly invoking such a skill via `/` still injects its content (explicit request wins).
- AC-4: Given an OpenClaw or Hermes session, the same rule applies to the folders each one natively reads — at per-folder granularity (a provider that reads only the workspace-level universal folder does not cause suppression of skills from the global one).
- AC-5: Every suppression decision is observable in session/harness diagnostics with its reason.

Edge cases:

- EC-1: The same skill also reachable from a Compozy root (shadow winner is Compozy) → injected normally; suppression keys on the winning root, not the name.
- EC-2: Provider without native reading of any enabled source (e.g. custom roots only) → nothing suppressed.
- EC-3: Session provider unknown/unset → no suppression (fail open to inclusion).
- EC-4: The session's provider home is relocated (environment override) or isolated away from the operator's home → the operator-home folders are no longer that session's native roots; their skills are injected normally.
- EC-5: A Hermes session with a skill whose winning copy lives in the global universal folder → injected (Hermes reads only the workspace-level universal folder natively).

## Expose

### US-010: Create a skill and expose it to other tools in one step

**As a** skill author, **I want** to create a skill and immediately expose it into enabled ecosystem folders, **so that** external tools see it under their conventions while one canonical copy lives with CompozyOS.

Acceptance criteria:

- AC-1: Given skill creation with an expose choice targeting the universal source, when it completes, then the canonical skill exists in the Compozy folder and a link exists at `.agents/skills/<name>` pointing to it.
- AC-2: Without an expose choice, creation behaves exactly as today (Compozy folder only).
- AC-3: An external tool reading the target folder resolves the link to the canonical content.

Edge cases:

- EC-1: Expose target source is not enabled → the operation fails with a message naming enabled targets; the skill itself is still created.
- EC-2: Target folder already has an entry with that name → expose fails naming the conflict; creation itself is unaffected.
- EC-3: Filesystem does not support links (or permission denied) → deterministic error; no silent copy fallback.
- EC-4: Target folder does not exist yet (enabled source, absent directory) → expose creates exactly that folder and succeeds; undoing a failed multi-target operation removes only folders the operation itself created and left empty.
- EC-5: Two expose operations for the same skill and target racing → exactly one wins; the other reports the conflict (or converges as already-exposed); no half-created state is ever visible.

### US-011: Expose/unexpose existing skills and trust cleanup

**As a** skill author, **I want** to expose or unexpose any eligible existing skill (including marketplace installs) and have lifecycle changes clean up after themselves, **so that** links never rot.

Acceptance criteria:

- AC-1: Given an eligible skill, when the author exposes it to a named enabled source, then the link is created and the skill's expose state is visible in skill inspection surfaces.
- AC-2: Unexpose removes the link and the state reflects it.
- AC-3: Removing a skill (including marketplace removal) removes every expose link that points at it.
- AC-4: Marketplace update preserves the skill's path so existing expose links stay valid.
- AC-5: Expose state is inspectable per skill: which sources it is exposed to, and whether each link is healthy.

Edge cases:

- EC-1: Exposing a bundled skill → rejected with a message explaining bundled skills have no on-disk home and pointing to the copy-via-create path.
- EC-2: Re-exposing to the same target → idempotent success (already exposed), not an error.
- EC-3: A link manually deleted outside CompozyOS → expose state reports it missing; re-expose repairs it.
- EC-4: A link whose target skill directory was moved manually → reported as broken in inspection surfaces with a repair action (unexpose or re-expose).
- EC-5: Unexposing a link that is not CompozyOS-created (same name, foreign symlink) → refused; CompozyOS only removes links that resolve into the skill it manages.
- EC-6: Removing a skill while one of its links cannot be cleaned (e.g. no permission on the target folder) → the removal aborts with a retryable error naming the failing link; the skill and its remaining state are preserved; re-running after fixing the cause completes cleanup and removal.
- EC-7: A path CompozyOS exposed later occupied by someone else's link or folder → inspection reports it as a foreign conflict with no repair action offered; CompozyOS never touches it.
- EC-8: Interruption mid-expose (crash between bookkeeping and link creation) → the exposure reads as missing afterwards; re-expose repairs it cleanly.
- EC-9: Running unexpose twice for the same target → second run succeeds as a no-op; repeated runs always converge to "not exposed".

## Agent management

### US-012: Inspect and change source config through structured surfaces

**As a** managing agent, **I want** structured commands and endpoints for every source operation, **so that** I can operate the feature without the web UI.

Acceptance criteria:

- AC-1: The agent can read active sources — enabled state, resolved paths, existence, per-source counts, truncation flags — as structured output at global and workspace scope.
- AC-2: The agent can set the preset list and custom list at both scopes through config-set semantics, and the response states apply semantics (live vs restart).
- AC-3: The agent can expose/unexpose eligible skills and read expose state per skill.
- AC-4: Every failure (unknown slug, invalid path, ineligible skill, disabled target) returns a deterministic, matchable error.

Edge cases:

- EC-1: Config write at agent scope → rejected; source keys are global/workspace policy (agent scope remains read-only for them).
- EC-4: Expose/unexpose invoked from an agent-scope session → allowed — exposing is a skill operation (like enable/disable), not source policy; it passes the same authorization gate as skill enable/disable.
- EC-2: Concurrent config writes to one scope → existing sequential-write discipline applies; last committed write wins; no interleaved partial state.
- EC-3: Structured outputs remain stable across HTTP and UDS (parity), byte-equivalent in field names.

## Settings UI

### US-013: Manage sources from Settings > Skills

**As an** operator, **I want** a sources section in Settings > Skills, **so that** I can see and change what CompozyOS scans without touching config files.

Acceptance criteria:

- AC-1: The section lists every known preset as a row — name, its folder pattern(s), enabled toggle, per-source skill count — with the Compozy row shown always-on without a toggle.
- AC-2: Rows reflect daemon truth: resolved paths, "directory absent" state, counts, truncation — never client-side assumptions.
- AC-3: A custom-sources editor supports adding and removing directory entries.
- AC-4: Scope control: global edits by default; workspace scope editable for the source keys with explicit "inherited vs overridden" indication (US-005).
- AC-5: Saving communicates the daemon-reported apply semantics (live) and refreshed counts appear without reload.

Edge cases:

- EC-1: Runtime unavailable → configuration remains editable; counts and existence states degrade explicitly instead of showing zeros.
- EC-2: All optional sources disabled and no custom entries → explicit "defaults only" presentation.
- EC-3: Agent scope selected → source section is read-only with the existing scope-policy notice.
- EC-4: A preset row whose directories don't exist → toggle still works; state communicates absence, not error.

## Diagnostics

### US-014: Truncation and name collisions are visible, never silent

**As an** operator, **I want** every scan limit hit and every dropped-name collision surfaced, **so that** "skill missing" is always diagnosable from product surfaces.

Acceptance criteria:

- AC-1: A root exceeding per-root scan limits is flagged as truncated in source inspection surfaces (settings, structured CLI/API output).
- AC-2: A skill name that cannot claim its bare command form (collision with builtin/agent command or another source's skill) is listed in diagnostics with its qualified fallback.
- AC-3: Skipped symlinks (escape, dangling, cycle) appear as per-entry diagnostics attributed to their root.
- AC-4: Per-root verification outcomes (how many definitions were blocked or warned) are visible alongside the counts, so "scanned 5, shows 3" is always explainable.

Edge cases:

- EC-1: Truncation resolves after the user trims the directory → flag clears on next refresh without restart.
- EC-2: Collision diagnostics at scale (dozens) → grouped/aggregated presentation, never one blocking error per item.

### US-015: Ecosystem skill definitions load without warning noise

**As an** operator, **I want** skills written for other tools (with their standard extra definition fields) to load without a wall of warnings, **so that** diagnostics stay meaningful when I enable ecosystem sources.

Acceptance criteria:

- AC-1: Given a skill authored for Claude Code (fields like tool restrictions, invocation hints, model preferences), when its source loads, then no warnings are emitted for those recognized fields — and CompozyOS does not act on them.
- AC-2: Given a definition with a truly unknown field, when it loads, then a warning still identifies the field — the signal survives.

Edge cases:

- EC-1: A recognized-but-inert field that looks like a permission grant (e.g. tool allowances) → never enforced by CompozyOS; documented as inert.
