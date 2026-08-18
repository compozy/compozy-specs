# User Stories: Agent Plugins Ingestion

Canonical behavior catalog for ingesting Agent Plugins packages (agent-plugins.org) as a third install layout of `compozy extension install`. Companion to `_spec.md`; consumed by `_spec.md` Part II (component mapping), `_uiux.md` (surface states), and `_tests.md` (coverage matrix).

Format vocabulary: an **Agent Plugins package** (or "portable package") is a directory with `plugin.json` at the root, optional `skills/<name>/SKILL.md` children, and an optional `mcp.json`. Once installed it **is an extension** — there is no separate product concept. This spec is about a package format, so format-level names (`plugin.json`, `SKILL.md`, `mcp.json`, transports `stdio`/`streamable-http`/`sse`) are the product surface.

## Personas

- **Operator** — runs Compozy, installs and manages content for their machine/workspaces; CLI-first, also uses the web UI (marketplace, inventory).
- **Agent** — a Compozy session operating extensions on the operator's behalf through CLI/native tools; needs structured output and deterministic errors for every action an operator can take.
- **Plugin author** — a developer shipping a portable package for the ecosystem (not necessarily Compozy-first); needs a trustworthy validator and a fast iteration loop against a live daemon.

## Story Index

| ID     | Feature Area      | Persona        | Story                                                        |
| ------ | ----------------- | -------------- | ------------------------------------------------------------ |
| US-001 | Install/Detection | Operator       | Install a conformant package from a local directory          |
| US-002 | Install/Detection | Operator       | Install from git/GitHub/archive sources                      |
| US-003 | Install/Detection | Operator       | Non-conformant or client-specific layout fails deterministically |
| US-004 | Install/Detection | Operator       | Dual-manifest directory resolves to the native format        |
| US-005 | Ingestion         | Operator       | Partial component failure is isolated and reported           |
| US-006 | Runtime delivery  | Operator       | Ingested skills work in sessions on every provider           |
| US-007 | Runtime delivery  | Operator       | Package MCP servers work in sessions with the env contract   |
| US-008 | Lifecycle         | Operator       | Enable/disable controls resource publication                 |
| US-009 | Lifecycle         | Operator       | Update refreshes content and preserves package data          |
| US-010 | Lifecycle         | Operator       | Remove deletes the instance and its data                     |
| US-011 | Lifecycle         | Operator       | Bind secrets for authenticated remote servers                |
| US-012 | Authoring         | Plugin author  | Validate a package directory against the standard            |
| US-013 | Authoring         | Plugin author  | Dev-link a package for live iteration                        |
| US-014 | Agent operations  | Agent          | Manage packages end-to-end with structured output            |
| US-015 | Observability     | Operator/Agent | Inventory and status expose format, resources, and degradation |
| US-016 | Marketplace       | Operator (web) | Discover and install a plugin-format catalog entry           |

## Install/Detection

### US-001: Install a conformant package from a local directory

**As an** operator, **I want** `compozy extension install <path>` to accept a directory whose root carries `plugin.json`, **so that** portable ecosystem content becomes a Compozy extension without any new command or concept.

Acceptance criteria:

- AC-1: Given a conformant package (valid `plugin.json`, at least the standard-required fields), when I install it, then the command succeeds and reports the package name, version (when present), detected format, and each ingested component (skills by name, MCP servers by name and transport).
- AC-2: Given a successful install, when I list extensions, then the new entry appears under the package's verbatim name with a format indicator, and it is inert (nothing published to sessions) until enabled.
- AC-3: Given a package name containing periods (`acme.tools`), when installed, then the identity is preserved byte-for-byte everywhere the name appears (list, status, remove, routes).
- AC-4: Given a minimal package (`plugin.json` with only the required fields, no skills, no MCP file), when installed, then the install succeeds with an empty resource set and the output says so plainly.
- AC-5: Given the install output, when I read it, then it never suggests a "plugin" is a different kind of thing than an extension — same lifecycle vocabulary throughout.

Edge cases:

- EC-1: `plugin.json` declares an unsupported or missing schema identifier → install fails with a deterministic error distinguishing "unsupported version of the Agent Plugins schema" from "not an Agent Plugins manifest".
- EC-2: Package name violates the standard's constraints (uppercase, edge hyphen/period, `--`, `..`, over 64 chars, empty) → install fails naming the violated constraint.
- EC-3: Manifest optional field carries an explicit wrong type (e.g., numeric `homepage`) → install fails (fatal per the standard); an unknown top-level field → install proceeds and the field is reported as ignored.
- EC-4: A package whose name collides with an already-installed extension → install fails with the existing-instance error; no partial state is left behind.
- EC-5: Package directory contains symlinks escaping the package root → the escaping entries are refused by containment hardening; outcome follows the ladder (fatal at manifest level, skip at component level) and is reported.
- EC-6: Archive source exceeding the size cap or containing traversal-shaped paths → install fails with the existing installer hardening errors; nothing is written.
- EC-7: Install interrupted (process killed mid-install) → no partially-registered extension is visible afterwards; re-running the install succeeds.
- EC-8: Two installs of the same package race → exactly one instance exists afterwards; the loser reports the existing-instance error.
- EC-9: A large package at AWS-kit scale (30+ skills, several MCP servers — roughly 10× the typical 2–5-skill package) → install completes and reports every component; no silent truncation of the component report.

### US-002: Install from git/GitHub/archive sources

**As an** operator, **I want** every source the installer already accepts (local path, archive, git/GitHub) to auto-detect the portable layout, **so that** where content lives never changes how it installs.

Acceptance criteria:

- AC-1: Given a GitHub/git source whose root carries `plugin.json`, when I install it, then detection, validation, ingestion, trust gating, and reporting behave exactly as the local-directory install.
- AC-2: Given an archive source, when the archive root (or its single wrapper directory) carries `plugin.json`, then the package installs like any archive-sourced extension.
- AC-3: Given a published (non-local) source, when the trust gate requires consent for unverified content, then the same consent flow applies — no format-specific trust bypass or extra prompt.

Edge cases:

- EC-1: Repository root carries the portable layout nested one wrapper directory deep (archive convention) → detection descends the single wrapper exactly as it does for native layouts.
- EC-2: Source is reachable but the fetched tree has no recognized manifest at root → deterministic "no recognized layout" error listing all three accepted root files.
- EC-3: Network failure mid-fetch → existing installer failure behavior; no partial instance.
- EC-4: The same package installed from two different sources under one name → second install fails with the existing-instance error; provenance keeps the first source of record.

### US-003: Non-conformant or client-specific layout fails deterministically

**As an** operator, **I want** a package that is *not* in the standard layout — a Claude Code plugin (`.claude-plugin/plugin.json`), a Codex-specific package (`.codex-plugin/`), or a repo with an unrelated root `plugin.json` — to fail with an error that tells me exactly what was found, **so that** the most common real-world miss is a clear answer instead of a mystery.

Acceptance criteria:

- AC-1: Given a directory carrying `.claude-plugin/plugin.json` (and no accepted root manifest), when I install it, then the error names the detected client-specific layout and points to the standard's migration path.
- AC-2: Given a root `plugin.json` whose schema identifier belongs to another ecosystem entirely, when no other accepted root manifest exists, then the error says the file is not an Agent Plugins manifest and lists the accepted root layouts.
- AC-3: Given any of these failures, when the command exits, then the failure is total — no extension instance, no partial resources, no data directory.

Edge cases:

- EC-1: Directory carries both `.claude-plugin/plugin.json` and a conformant root `plugin.json` → the conformant root layout wins; the client-specific directory is ignored inert content.
- EC-2: Root `plugin.json` is a symlink or non-regular file → refused by containment; treated as no manifest present.
- EC-3: Agent installing (not human) → the same failure arrives as a deterministic, machine-readable error — no interactive-only guidance.

### US-004: Dual-manifest directory resolves to the native format

**As an** operator, **I want** a directory carrying both `extension.toml` and `plugin.json` to install as a native extension, **so that** authors can dual-target one directory and Compozy always prefers its richer native manifest.

Acceptance criteria:

- AC-1: Given a directory with both manifests, when installed, then the native manifest governs entirely and the portable manifest is inert content; the install output notes the dual-manifest resolution.
- AC-2: Given a directory with `SKILL.md` and `plugin.json` at root, when installed, then a single deterministic precedence applies and is stated in the output (no error, no ambiguity).

Edge cases:

- EC-1: The native manifest is invalid while the portable one is valid → the install fails on the native manifest's error (precedence is decided before validation; no silent fallback to the other format).
- EC-2: Enable/disable/update/remove on a dual-manifest install → behave as pure native-extension operations; the portable manifest stays inert forever.

## Ingestion

### US-005: Partial component failure is isolated and reported

**As an** operator, **I want** a package with some broken components to install its healthy ones and tell me exactly what was skipped and why, **so that** one bad skill never takes down a 30-skill kit.

Acceptance criteria:

- AC-1: Given a package where one skill's `SKILL.md` is invalid (bad frontmatter, name/directory mismatch, missing required fields), when installed, then that skill is skipped with a per-skill reason and every valid sibling is ingested.
- AC-2: Given one invalid MCP server entry among valid ones, when installed, then the invalid entry is skipped with a per-server reason and the siblings survive.
- AC-3: Given an `sse`-transport server entry, when installed, then the entry is skipped with a distinct "transport not supported" message — and a *malformed* `sse` entry reports "invalid entry" instead, so a broken entry cannot masquerade as merely unsupported.
- AC-4: Given the standard's `extensions` namespace object or client-owned namespace directories (`com.example.client/`), when installed, then they are ignored without validating their contents and without diagnostics noise beyond a factual note.
- AC-5: Given any skip, when I inspect install output or the extension's status afterwards, then the skip and its reason are visible (ADR-003) — not log-only.

Edge cases:

- EC-1: The `skills` location exists but is a file, not a directory → the skills component type is disabled for this package with a reason; MCP ingestion continues.
- EC-2: `mcp.json` exists but is unreadable/malformed as a whole → the MCP component type is disabled with a reason; skills continue.
- EC-3: `mcp.json` schema identifier is unsupported → MCP disabled for the package with a reason; skills continue (component-level, not fatal — asymmetric with the root manifest on purpose).
- EC-4: A skill nested deeper than one level (`skills/group/nested/`) → not discovered at all (the standard's non-recursive rule); no diagnostic required, but the ingested-component report makes the absence visible.
- EC-5: Every component of a package is invalid → the install still succeeds as an empty, degraded instance with all reasons visible; the operator can remove it or fix upstream.
- EC-6: Duplicate MCP server names after normalization → deterministic error for the colliding entries, survivors keep loading.
- EC-7: A server entry named to collide with reserved words (e.g., a server literally named `mcpServers`) → treated as an ordinary name; no key-namespace confusion.

## Runtime delivery

### US-006: Ingested skills use the normal session resource path

**As an** operator, **I want** an enabled package's skills to behave exactly like any extension-shipped skills in sessions, **so that** portable content does not need provider-specific installation.

Acceptance criteria:

- AC-1: Given an enabled package with valid skills, when a supported managed session starts, then those skills are available under the standard skill activation flow, indistinguishable in use from native extension skills. The complete skill-plus-MCP path is verified on Claude Code and Hermes.
- AC-2: Given skill content, when it is loaded, then the existing load-time content security scan applies with its normal blocking/logging behavior.
- AC-3: Given the package is disabled, when a new session starts, then its skills are absent.

Edge cases:

- EC-1: A skill name collides with a same-named skill from another extension → the existing extension-skill precedence/shadowing rules apply and log the shadow; no new policy.
- EC-2: Skill references sidecar files (`references/`, `scripts/`) inside its own directory → they resolve inside the package root; anything resolving outside the root is refused.
- EC-3: A session is already running when the package is enabled → the running session is unaffected; the next session sees the skills (existing kit-publication semantics).

### US-007: Package MCP servers work in sessions with the env contract

**As an** operator, **I want** a package's declared MCP servers (`stdio` and `streamable-http`) reachable in sessions with the standard's runtime contract honored, **so that** tools shipped by portable content actually run.

Acceptance criteria:

- AC-1: Given an enabled package with a `stdio` server, when a session uses it, then the process launches with `PLUGIN_ROOT` and `PLUGIN_DATA` set to absolute paths, `${PLUGIN_ROOT}`/`${PLUGIN_DATA}` expanded in args, env values, and working directory — and nowhere else (never in the command token).
- AC-2: Given a `streamable-http` server, when a session uses it, then the connection targets the declared URL with the declared headers, subject to the URL policy (absolute http/https, no userinfo/fragment, plain HTTP only for loopback).
- AC-3: Given the package's data directory, when servers write to it, then contents persist across daemon restarts and package updates.
- AC-4: Given a server's declared env, when the process launches, then the reserved keys (`PLUGIN_ROOT`, `PLUGIN_DATA`) cannot be overridden by the package, and expansion is single-pass (an expanded value containing a placeholder is not re-expanded).
- AC-5: Given a provider whose ACP bridge rejects per-session MCP, when the package requires hosted MCP, then session creation fails before provider launch with a clear unavailable-capability error. OpenClaw is outside the portable MCP delivery promise until its ACP bridge accepts the required session resources.

Edge cases:

- EC-1: Host environment variables (`${AWS_SECRET}` and friends) referenced in package config → never interpolated; unrecognized placeholder text stays literal (deliberate asymmetry with native config).
- EC-2: The command token is a bare name not present on PATH → the server fails at launch with the normal MCP failure behavior; the failure is per-server, visible in diagnostics, and never blocks the session or sibling servers.
- EC-3: Working directory resolves outside its declared containment root → that server was already skipped at ingestion with a containment reason.
- EC-4: Data directory cannot be created at first stdio launch (permissions, read-only home) → that server reports live unhealthy state with the reason; skills, remote servers, and sibling stdio servers are unaffected; install/update never fail for this.
- EC-5: A package server name collides with a session-level MCP server from config → deterministic precedence with a visible resolution (no silent double-registration).

## Lifecycle

### US-008: Enable/disable controls resource publication

**As an** operator, **I want** the standard extension enable/disable lifecycle to govern ingested packages, **so that** installing is safe-by-default and activation is a deliberate act.

Acceptance criteria:

- AC-1: Given a fresh install, when I enable the extension, then its skills and MCP servers publish to new sessions; when I disable it, they unpublish for new sessions.
- AC-2: Given enable/disable operations, when performed via CLI, API, or web, then behavior and reported state agree across all three surfaces.

Edge cases:

- EC-1: Enabling a fully-degraded (empty) install → succeeds; status shows enabled-with-zero-resources plus the skip reasons.
- EC-2: Disable while sessions are running → running sessions keep their already-merged resources; new sessions exclude them (existing semantics, no format-specific behavior).
- EC-3: Enable/disable racing with update → operations serialize per instance; the surviving state is one of the two requested states, never a hybrid.

### US-009: Update refreshes content and preserves package data

**As an** operator, **I want** updates for plugin-format installs to follow the existing extension update flow, **so that** there is exactly one update story.

Acceptance criteria:

- AC-1: Given an installed package from a git/GitHub source, when the upstream content changes and I update, then the new content is validated and ingested like a fresh install (same ladder, same visibility) and the instance keeps its identity.
- AC-2: Given an update, when it completes, then the package's persistent data directory is preserved bit-for-bit (the standard's requirement).
- AC-3: Given the package's `version` field, when displayed, then it is metadata only — the update decision is content-driven, not version-driven.

Edge cases:

- EC-1: Upstream restructures away from the standard layout → update fails with the layout diagnostic; the installed instance stays on its previous content, still functional.
- EC-2: Update introduces new invalid components → healthy components refresh, invalid ones are skipped with visible reasons (ladder applies on update, not just install).
- EC-3: Update while the extension is disabled → content refreshes; enabled state is preserved as-is.
- EC-4: No upstream change → update reports no-op; nothing rewritten.

### US-010: Remove deletes the instance and its data

**As an** operator, **I want** `compozy extension remove` to delete the ingested extension and its persistent data directory, **so that** removal leaves no orphaned state (both reference clients orphan data — Compozy does not).

Acceptance criteria:

- AC-1: Given an installed package with a data directory, when I remove it, then the instance, its registration, and its data directory are gone; a subsequent identical install starts clean.
- AC-2: Given the removal output, when I read it, then it states the data directory was removed.

Edge cases:

- EC-1: Remove while enabled → resources unpublish for new sessions and the instance is removed (existing remove semantics; no format-specific gate).
- EC-2: Data directory already absent (never created — no `stdio` servers ever ran) → remove succeeds without noise.
- EC-3: Data directory deletion fails (filesystem error) → the residue is quarantined under a non-reusable name and removal completes with the quarantine path reported; a later identical install starts with an empty data directory. If even quarantine fails, the removal fails deterministically and the instance stays.
- EC-4: Remove interrupted → re-running remove converges to fully-removed; no half-registered state.

### US-011: Bind secrets for authenticated remote servers

**As an** operator, **I want** to bind secrets to an ingested package through the existing extension secret-binding mechanic, **so that** authenticated `streamable-http` servers work without ever editing package files or embedding credentials.

Acceptance criteria:

- AC-1: Given a package whose remote server requires a credential, when I bind a secret to the extension by name, then sessions using that server receive the credential through the binding — the package files remain untouched.
- AC-2: Given any public read (status, inventory, API), when secrets are bound, then only bound key names appear — never values or references.
- AC-3: Given package-declared header/env values, when ingested, then they are treated as visible package data, never as secrets; a package-declared sensitive header (authorization and its equivalents) causes that server entry to be refused with a visible scoped diagnostic — never silently stripped.

Edge cases:

- EC-1: Binding referencing a nonexistent vault entry → the existing binding error surfaces; server behavior on missing credential is a connection failure, not invalid config (standard rule).
- EC-2: Removing the extension → its bindings are removed with the instance.
- EC-3: A package attempts to declare reserved or sensitive headers directly (e.g., an authorization header with a literal token) → that server entry is refused with a scoped diagnostic (never silently stripped); the binding path is the only credential path.

## Authoring

### US-012: Validate a package directory against the standard

**As a** plugin author, **I want** `compozy extension validate <path>` to run full standard conformance on my package, **so that** I can trust Compozy's verdict before shipping — there is no official validator in the ecosystem.

Acceptance criteria:

- AC-1: Given a package directory, when I validate it, then fatal manifest errors and every per-component diagnostic are reported with their scope (which skill, which server), in both human and structured output.
- AC-2: Given a fully conformant package, when validated, then the result is an explicit pass listing what would be ingested.
- AC-3: Given validation, when it runs, then nothing is installed, no data directory is created, and no daemon state changes.
- AC-4: Given exit behavior, when fatal errors exist, then the exit is failure; diagnostics-only results exit success with the diagnostics listed (matching the install-time ladder).

Edge cases:

- EC-1: Validate a native extension directory → existing native validation runs; format detection is automatic, same precedence as install.
- EC-2: Validate a dual-manifest directory → validates as native (precedence rule), with the resolution noted.
- EC-3: Validate a client-specific layout → same deterministic layout error as install (US-003), making validate the cheap pre-flight for "will this install?".

### US-013: Dev-link a package for live iteration

**As a** plugin author, **I want** to dev-link a portable package directory into a workspace, **so that** I can edit, reload, and test against live sessions without reinstalling.

Acceptance criteria:

- AC-1: Given a dev link to a package directory, when created, then the package ingests with dev-link scoping (workspace instance) and the normal ladder visibility.
- AC-2: Given an edit to the linked directory (fix a skill, add a server), when reloaded, then the changes take effect under the same validation, and new diagnostics surface immediately.

Edge cases:

- EC-1: Edit introduces a fatal manifest error → reload fails with the fatal error; the previous good state remains active until a valid reload.
- EC-2: Dev-linked and marketplace-installed copies of the same name coexist → existing precedence between workspace and global instances applies and is visible in status.

## Agent operations

### US-014: Manage packages end-to-end with structured output

**As an** agent, **I want** every operation of this feature (install, validate, enable, disable, update, remove, status, inventory, secrets) available through the CLI/native tools with structured output and deterministic errors, **so that** I can operate portable content autonomously without the web UI.

Acceptance criteria:

- AC-1: Given any lifecycle operation, when performed with structured output, then the result carries machine-readable fields for format, ingested components, skipped components with scoped reasons, and final state.
- AC-2: Given any failure (layout, fatality ladder, existing-instance, trust), when it occurs, then the error is deterministic and distinguishable programmatically — an agent can branch on it without parsing prose.
- AC-3: Given the operations, when performed via HTTP/UDS instead of CLI, then state and reported fields agree with the CLI (surface parity).

Edge cases:

- EC-1: Agent installs a package requiring unverified-source consent in a non-interactive context → the operation fails deterministically with the consent-required error; no hang, no silent bypass.
- EC-2: Agent queries a name that is not installed → deterministic not-found error, identical shape across CLI/HTTP/UDS.
- EC-3: Concurrent agent operations on one instance → serialized per instance; second operation sees a deterministic busy/conflict outcome or the winner's final state, never corruption.

## Observability

### US-015: Inventory and status expose format, resources, and degradation

**As an** operator or agent, **I want** the format, the synthesized resources, and any degraded components visible in `status`, `inventory`, list output, and the web detail, **so that** an ingested package's health is inspectable without reading logs.

Acceptance criteria:

- AC-1: Given an ingested package, when I view its status (human and structured), its inventory, list structured output, or the web detail/cards, then the format indicator is present (read-only badge — never a different lifecycle); the human list table keeps its existing columns by decision.
- AC-2: Given skipped components, when I view status/inventory, then each skip appears with its scope and reason, persisted from install/update time.
- AC-3: Given provenance output, when I inspect it, then source, trust tier, and content digest appear exactly as they do for native extensions.

Edge cases:

- EC-1: Zero-resource (empty or fully-degraded) install → surfaces render an explicit empty state with reasons, not a blank.
- EC-2: Format indicator on native extensions → absent (indicator marks the exception, not the rule).
- EC-3: Web and CLI disagree-check → the same underlying state produces consistent fields on both (parity is testable).

## Marketplace

### US-016: Discover and install a plugin-format catalog entry

**As an** operator (web), **I want** curated catalog entries that point at portable packages, with a format badge, installable through the normal marketplace flow, **so that** I can discover ecosystem content without hunting GitHub.

Acceptance criteria:

- AC-1: Given a curated feed entry marked as plugin-format, when I browse the marketplace, then the card renders with the format badge alongside the normal metadata.
- AC-2: Given the entry, when I install from the web, then the standard install flow runs (trust dialog, unverified consent when applicable) and the result matches a CLI install of the same source.
- AC-3: Given the installed entry, when I view it in installed/inventory surfaces, then badge and component data appear per US-015.

Edge cases:

- EC-1: A curated entry whose upstream drifted non-conformant → the web install fails with the same deterministic layout diagnostic; the failure is legible in the dialog, not a toast mystery.
- EC-2: Feed entry without the format marker pointing at a portable package → install still works (detection is authoritative; the marker is display metadata).
- EC-3: Marketplace-tier ceiling interaction → a resource-only synthesis requests no permissions, so consent shows resource publication only — never a permission grant list.
