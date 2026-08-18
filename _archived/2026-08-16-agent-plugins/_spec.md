# Spec: Agent Plugins Ingestion

One spec, two parts. Part I frames the product (WHAT/WHY/WHO); Part II (written after the Stage 1 checkpoint) designs the implementation. Companions: [_user_stories.md](_user_stories.md), `_dx.md`, `_uiux.md`, `_tests.md`, `adrs/`.

**Format-naming exception (L-013):** this spec is *about* a third-party package format. The format's own artifact names (`plugin.json`, `SKILL.md`, `mcp.json`) and transport labels (`stdio`, `streamable-http`, `sse`) are the user-observable product surface and appear in Part I by necessity. Compozy-internal implementation choices stay out of Part I.

---

# Part I — Product

## Overview

The Agent Plugins standard (agent-plugins.org, v1.0.0, launched 2026-08-06 by a Vercel-initiated TSC with AWS, Cursor, GitHub/Microsoft, OpenAI, and Google) defines a portable package for agent content: a plain directory with a `plugin.json` manifest, skills under `skills/<name>/SKILL.md` (the agentskills.io format), and MCP servers in `mcp.json`. Nine clients already consume it — including OpenClaw (full conformance) and Hermes (explicit compatibility-adapter subset), two of the three ACP providers Compozy spawns. AWS, Google, and Vercel are shipping content for it.

Compozy cannot install this content today: the installer recognizes only its two native root layouts. Operators who find ecosystem content have no path to use it; agents cannot acquire it; authors have no way to check their packages against Compozy — and the ecosystem itself has no official validator at all.

This feature makes `compozy extension install` accept the portable layout as a **third install layout** — auto-detected, validated against the standard, and synthesized into a normal resource-only extension. No new product concept, no new verb: once installed, the package **is an extension** with the full existing lifecycle (enable/disable/update/remove/secrets/trust/provenance). Delivery uses the daemon-owned extension path instead of provider-specific package installation. Real-provider QA proves the complete skill and MCP path on Claude Code and Hermes. OpenClaw's ACP bridge rejects per-session MCP servers, so a package that requires hosted MCP fails before provider launch instead of starting without its declared tools. Compozy layers on the things the standard deliberately lacks: a trust gate, content scanning, provenance, and operator-bound secrets.

Who it is for: **operators** installing ecosystem content, **agents** operating that content autonomously through structured surfaces, and **plugin authors** who need a trustworthy conformance verdict and a live iteration loop.

## Goals

- Operators can install any conformant Agent Plugins package from every source the installer already accepts (local directory, archive, git/GitHub, dev link) with format auto-detection — no new command, flag, or concept.
- Ingested packages behave as first-class extensions: the entire existing lifecycle applies unchanged, and their skills and MCP servers use the native extension-resource path on providers that accept the required session resources. Claude Code and Hermes are the verified delivery targets; OpenClaw's ACP bridge is an explicit MCP limitation.
- The standard's runtime contract holds for ingested content: package-scoped persistent data survives updates, the two portable placeholder variables expand exactly where the standard says and nowhere else, and path containment is enforced.
- Conformance outcomes are first-class product output (ADR-003): install, validate, status, and inventory expose every fatal error and every skipped component with a scoped reason — visible to humans and structured for agents. Both reference clients bury these in logs; Compozy does not.
- Plugin authors get a real conformance tool: `compozy extension validate` runs the full standard ladder on a package directory — filling a gap the ecosystem itself has not filled (no official validator exists).
- Agents can perform every operation of this feature through CLI/native tools with structured output and deterministic errors, with HTTP/UDS parity (SD-011).
- Plugin-format content is discoverable in the marketplace: curated feed entries may point at portable packages, rendered with a read-only format badge and installable through the normal flow (ADR-002).
- Compozy qualifies for the standard's compatible-clients table: the 8-item minimum-conformance checklist passes, including both `stdio` and `streamable-http`. The external listing remains a post-merge follow-up owned by the user so it can link to the deployed interoperability page.

## User Stories

Canonical catalog: [Full user stories](_user_stories.md).

- US-001..US-004 — Install & detection: local/git/archive install, deterministic non-conformant failures, dual-manifest precedence.
- US-005 — Ingestion: component-level failure isolation with visible reasons.
- US-006..US-007 — Runtime delivery: skills and MCP servers in sessions across providers, env contract honored.
- US-008..US-011 — Lifecycle: enable/disable, checksum-driven update with data preservation, clean remove, secret bindings.
- US-012..US-013 — Authoring: full-conformance validate, dev-link iteration loop.
- US-014 — Agent operations: structured output + deterministic errors + surface parity.
- US-015 — Observability: format badge, resource inventory, degradation reasons.
- US-016 — Marketplace: curated plugin-format entries with badge and normal install flow.

## Core Features

- **Format auto-detection at install.** The installer recognizes a third root layout (`plugin.json`) beside the two native ones. Detection is by root manifest with a deterministic precedence: a native manifest in the same directory always wins (dual-target directories are legal — the standard closes manifest fields, not directory contents). A root `plugin.json` from another ecosystem is distinguished from an unsupported standard version, and client-specific layouts (`.claude-plugin/`, `.codex-plugin/`) fail with an error naming what was found and pointing at the standard's migration path (US-003; no heuristic fallback — a Claude Code adapter is a separate future effort).
- **Standard-conformant ingestion.** Validation implements the standard's fatality ladder: fatal manifest errors reject the package; component-level problems skip that component with a scoped reason and never poison siblings. Skills are discovered non-recursively (immediate children of `skills/` only, the standard's rule). `stdio` and `streamable-http` servers are ingested; `sse` entries are shape-checked then skipped with a distinct "unsupported transport" diagnostic (a malformed entry reports "invalid" instead — two different problems, two different messages). The `extensions` namespace object and client-owned namespace directories are ignored without content validation, as the standard mandates.
- **Resource-only synthesis.** A validated package becomes a normal extension instance carrying only resources (skills + MCP server declarations): no extension runtime process, no host-permission grants — a strictly smaller risk surface than a typical extension, which is why it fits every trust tier including the marketplace ceiling.
- **Portable runtime contract.** Each ingested instance gets a persistent package data directory (created when needed, preserved bit-for-bit across updates, deleted on remove — ADR: divergence from both reference clients, which orphan it). Sessions launching a package's `stdio` server receive the two standard variables as absolute paths; placeholder expansion is single-pass and applies only to arguments, environment values, and working directory — never the command token, never host environment interpolation into package config.
- **First-class diagnostics (ADR-003).** One recorded diagnostic set per instance feeds install output, validate output, status, inventory, and the web detail. Every diagnostic carries a component scope. Agents get the same data structured.
- **Conformance validator for authors.** `compozy extension validate <path>` auto-detects the layout and runs the identical ladder with deterministic exit behavior — pass, diagnostics-only, or fatal — without touching daemon state.
- **Marketplace surfacing (ADR-002).** Curated feed entries may declare the plugin format; cards and detail views render a read-only badge; web install runs the same trust-gated flow as any marketplace entry.
- **Compatible-clients listing handoff (Phase 2, trailing task).** The workstream records the current upstream shape, per-transport evidence, and deployed-doc prerequisite. The user owns the external PR after this CompozyOS change is merged and the interoperability page is live.

Feature interactions: detection feeds ingestion; ingestion feeds both the instance synthesis and the diagnostic set; the diagnostic set feeds every observability surface; the synthesis feeds the unchanged lifecycle; the lifecycle feeds runtime delivery; validate reuses detection + ingestion in a stateless mode; the marketplace rides detection + trust unchanged.

## Business Rules

Identity and naming:

1. The installed identity is the package's manifest name, byte-for-byte, everywhere (ADR-004). Package names follow the standard's constraints: 1–64 characters, lowercase letters/digits/periods/hyphens, alphanumeric first and last character, no consecutive hyphens or periods. Violations are fatal.
2. One instance per name: installing over an existing name fails with the existing-instance error; no silent replace.
3. "Plugin" is never a product noun on any Compozy surface. The format indicator is a read-only badge; every lifecycle word is the extension vocabulary.

Detection and precedence:

4. Root-layout precedence is a fixed matrix decided before validation: both native manifests at one root stays the existing hard error (regardless of `plugin.json`); exactly one native root wins over `plugin.json`; a portable package is selected only when no native root exists; client-specific layouts never participate; a selected invalid root fails the install rather than falling through.
5. A root `plugin.json` whose schema identifier is not the supported standard version is classified as either "unsupported Agent Plugins version" (standard-prefixed identifier) or "not an Agent Plugins manifest" (anything else), each with its own deterministic error.
6. Client-specific plugin layouts are never ingested; the failure names the detected layout and points to the standard's migration guide.

Ingestion ladder (user-facing outcomes):

7. Fatal (whole package rejected, nothing registered): missing/invalid schema identifier, missing/invalid/constraint-violating name, wrong-typed manifest fields, explicit-null optional fields, unreadable manifest.
8. Non-fatal (reported, ingestion continues): unknown top-level manifest fields (ignored + reported), non-object `extensions` (dropped + reported), any skills-location or per-skill problem (skip with reason), any `mcp.json` or per-server problem (skip with reason). The `mcp.json` schema version must match the `plugin.json` schema version — a mismatch disables the MCP component for that package (with a reason) and never affects skills.
9. A skill is skipped when its `SKILL.md` is missing/invalid or its declared name does not match its directory name (the skills format's rule). Skill discovery never recurses past immediate children of `skills/`.
10. An MCP server entry is skipped when its shape is invalid, its transport is unknown, it is `sse`, its command token is not a bare executable name or a package-relative path, its working directory escapes its containment root, or it attempts to override the two reserved variables.
11. Component skips are recorded with scope + reason and remain visible on status/inventory surfaces for the life of the install (until an update clears them).
12. An empty or fully-degraded package still installs (the standard treats missing locations as non-errors); surfaces render the empty state with reasons explicitly.

Trust, security, and secrets:

13. The existing trust pipeline applies unchanged: source tiers, unverified-source consent, content security scan on skills, content digest, and provenance record. No format-specific policy, prompt, or bypass exists.
14. A synthesized instance is resource-only: it never requests host permissions, so consent surfaces show resource publication only.
15. Packages cannot carry working credentials: declared header/env values are visible package data, and a package-declared sensitive header (authorization and its equivalents) causes that server entry to be refused with a visible scoped diagnostic — never silently stripped. Authenticated remote servers get credentials exclusively through operator-bound extension secrets, whose public reads expose key names only.
16. Remote server URLs must be absolute http/https without userinfo or fragment; plain HTTP is allowed only for loopback hosts.
17. Every path read from a package must resolve inside the package root after symlink resolution; escaping entries are refused per the ladder.

Lifecycle:

18. Install leaves the kit inert; enable publishes resources to new sessions; disable unpublishes for new sessions; running sessions are never mutated by lifecycle operations.
19. Update follows the existing content-driven (checksum) flow; the manifest `version` field is display metadata only. Updates re-run the full ladder; the data directory is preserved bit-for-bit across updates.
20. Remove deletes the instance, its registration, its secret bindings, and its data directory; a failed data-directory deletion still completes the removal and reports the leftover path.
21. Operations on one instance serialize; concurrent operations converge to a deterministic outcome, never a hybrid state.

Marketplace:

22. Feed format markers are display metadata; install-time detection is authoritative. A drifted upstream fails install with the standard layout diagnostic.

## User Experience

Personas and goals: see [_user_stories.md](_user_stories.md) Personas. Journeys:

- **Operator golden path:** find a package (marketplace card with badge, or a repo URL) → `compozy extension install <source>` or web install → read the ingestion report (components in, skips with reasons) → `compozy extension enable <name>` → start a session and use the skills/tools. First contact to working content in two commands.
- **Operator repair path:** a package misbehaves → `status`/`inventory` show degraded components with reasons → fix is either upstream (report to author), an update, or removal. No log spelunking.
- **Author loop:** write the package per the standard → `compozy extension validate ./pkg` until clean → dev-link into a workspace → edit/reload/test in live sessions → ship. Compozy is the conformance tool the ecosystem lacks.
- **Agent path:** an agent discovers, installs (consent rules permitting), inspects, enables, and repairs packages entirely through structured CLI/native-tool calls with deterministic errors — no web UI dependency (SD-011).
- **Web path:** marketplace browse → badge signals format → install dialog with the normal trust flow → installed card and detail views show badge, resources, and any degradation.

Discoverability: the feature is invisible until relevant — no new commands to learn; the badge and the ingestion report are the only new things an operator sees. Accessibility: web surfaces follow the existing design-system rules; no new interaction patterns are introduced.

## High-Level Technical Constraints

- **Provider-neutral delivery with an explicit capability boundary:** ingested content uses the same daemon-owned resource path wherever the provider accepts the required session resources. Claude Code and Hermes must pass the complete real-provider walk. A provider that rejects session MCP must fail before launch with a clear error and stay outside the portable MCP delivery claim.
- **Standard conformance is the definition of done:** the 8-item minimum-conformance checklist (standard §11.1) must pass, with both portable transports supported. Conformance claims must be pinned to the exact schema version supported.
- **Agent/operator manageability outcome:** every capability of this feature is inspectable, operable, and repairable from the CLI and machine surfaces with structured output and deterministic errors; HTTP/UDS state parity where daemon state crosses the boundary. The web UI is a convenience, never the only path (SD-011).
- **Extension ecosystem expectation:** this feature *is* an extension-surface change — it must slot into the existing extension install/trust/lifecycle/registry machinery rather than beside it, and it must not add runtime extension points of its own.
- **No new configuration:** format detection is not policy; the existing trust configuration governs. Zero new config keys (closed decision).
- **Security floor:** existing installer hardening (size caps, traversal/symlink refusal, content scan, digest, provenance) applies to the new layout unchanged; the standard's containment and no-credential rules are enforced on top.
- **Additive change:** no existing behavior, storage, API, or command is removed or renamed. **Delete targets: none** — stated per the greenfield rule; this spec introduces a new layout without breaking any existing contract.
- **Performance from the user's seat:** install/validate of a typical package (tens of skills) completes interactively; the component report never truncates silently.

## Non-Goals (Out of Scope)

- **No separate "plugin" product concept** — no new noun, verb, registry, or UI taxonomy (ADR-001).
- **No export/dual-target (Phase 3):** `compozy extension publish` emitting the portable subset under a `com.compozy/` namespace is explicitly deferred.
- **No Claude Code plugin adapter:** `.claude-plugin/` packages (a richer, renamed superset) fail with guidance; ingesting them is a possible later effort with its own spec.
- **No `sse` transport support:** legacy entries are skipped with a diagnostic (spec-optional for clients; the deprecated MCP transport).
- **No dedicated marketplace section:** plugin-format entries live in the existing curated feeds only (ADR-002); no automatic ecosystem crawling or third-party registry integration.
- **No new configuration keys** — trust policy reuses the existing unverified-content setting and consent pipeline.
- **No signing/attestation invention:** the standard defers provenance; Compozy applies its existing provenance/digest machinery and goes no further in this phase.
- **No re-platforming of the native manifest** onto the portable format (rejected alternative, ADR-001).

## Open Questions

None at the product level — every product fork raised in the grill rounds was resolved (Rounds 1–2, 2026-08-15). Technical items carried into Stage 2: package data-directory location and layout; synthesis of the daemon-version compatibility gate for adapted manifests; the E2E fixture package choice; catalog feed field shape; diagnostic persistence shape.

---

# Part II — Technical

## Executive Summary

Ingestion is an adapter, not a subsystem: a new pure conformance package reads and validates an Agent Plugins directory, and the extension manifest loader converts its output into a normal resource-only manifest **in memory on every load** (ADR-005) — `plugin.json` stays the only manifest on disk, the installed tree stays byte-identical to upstream, and everything downstream (registry, kit publication, session delivery, trust, secrets) is existing machinery. Two deliberate schema widenings ship with it: the extension-manifest MCP declaration gains `transport`/`url`/`headers` (today stdio-only), and the canonical session `MCPServer` gains `Headers`/`SecretHeaders` consumed exclusively by the daemon's own MCP executor client (ADR-006) — remote servers never serialize to ACP providers, so package credentials never leave the daemon. Persistence adds instance-scoped `format` + ingest-diagnostics columns to both instance owners (`extensions`, `extension_dev_links`) and the remote-header mapping to `extension_env_bindings`; package data gets a dedicated `extension-data/` root that cannot overlap managed installs. No new CLI verbs, HTTP/UDS routes, config keys, or product concepts.

## MVP Boundary

MVP boundary: every numbered implementation task derived from this spec except the trailing tasks composes the MVP — conformance reader, installer detection, synthesis + persistence, MCP header widening, lifecycle (data dir, update, remove), secrets remote-header binding, CLI/API/native-tool projections, web badge + skipped rows, catalog feed field, docs and official-skill updates. The trailing tasks run in evidence order: `qa-report`, `qa-execution` (provider matrix + conformance walk), then the Phase 2 compatible-clients handoff conditional on that recorded evidence. The matrix proves Claude Code and Hermes delivery and records OpenClaw's ACP limitation without treating it as a pass. Explicitly out of scope: Phase 3 export/dual-target, a Claude Code plugin adapter, an OpenClaw gateway-side MCP bridge, `sse` transport support, any dedicated marketplace section, and exposing the MCP header fields on operator config surfaces (sidecar/config verbs/web editor — future spec).

## Developer Experience

- [Developer experience contract](_dx.md) — golden path, package layout, CLI verb-by-verb transcripts, HTTP/UDS payloads, catalog feed entry, native tools, deterministic error table.
- [UI change map](_uiux.md) — marketplace card/detail badges, extension badge variant, kit-inventory skipped rows, dialogs explicitly unchanged.

## System Architecture

| Component | Responsibility |
| --- | --- |
| `internal/extension/agentplugin` (new) | Standard-conformance layer: `$schema` triage, fatality ladder, non-recursive skill discovery, MCP entry validation/translation, placeholder expansion, scoped diagnostics. No I/O beyond the package directory. Allowed imports: stdlib + `internal/frontmatter` (YAML frontmatter parse) + `internal/mcppolicy` (shared header/URL policy) — nothing else. Production file split fixed up front: `types.go` (contracts + diagnostics), `classify.go` ($schema triage), `manifest.go` (plugin.json ladder), `skills.go` (discovery + agentskills frontmatter conformance), `mcp.go` (mcp.json + per-entry translation), `paths.go` (containment + expansion). |
| `internal/mcppolicy` (new leaf) | Single authority for MCP header/URL policy shared by plugin ingestion, sidecar validation, and the executor: RFC 7230 token names, case-insensitive duplicate detection across fixed+secret maps, the sensitive-header set, source-aware credential rules, and the remote-URL policy. Stdlib only. |
| `internal/registry` installer (modified) | Third accepted root (`plugin.json`) with tri-state schema triage; existing hardening (size caps, traversal/symlink refusal, content scan, digest) unchanged. |
| `internal/extension` (modified) | Manifest-load branch → synthesis (ADR-005); `min_compozy_version` waiver for adapted manifests; `format` + ingest-diagnostics persistence; data-directory lifecycle on install/update/remove; manifest `MCPServerConfig` widening. |
| `internal/config` (modified) | Canonical `MCPServer` gains `Headers`/`SecretHeaders` as **internal wire fields**: produced by plugin synthesis and secret-binding projection, consumed by the daemon executor, validated via `mcppolicy`, redacted in every read. They are **not** exposed on the workspace `mcp.json` sidecar, `compozy config` MCP verbs, or the web MCP editor in this spec — that generalization is a future spec with its own frozen surface (partial-surface cut, round-2 B-005). |
| `internal/mcp` executor client (modified) | Resolves `SecretHeaders` references immediately before request construction, injecting values only into a non-serializable `http.Header` through the existing `RoundTripper` seam; owns the live per-server runtime-health state (launch/auth/connection failures) keyed by extension instance + server name, projected as live diagnostics. |
| Secrets bindings (modified) | `secrets bind --remote-header <server>:<header>` persists the mapping; session assembly projects the binding as a **reference** into that server's `SecretHeaders` — only the MCP executor resolves it, immediately before request construction, into a non-serializable request header. |
| `internal/marketplace` + web (modified) | Feed entry optional `format` field (hard cut at the current feed version); format badge, skipped-components section per `_uiux.md`. |
| CLI/API projections (modified) | `format` + `diagnostics` on extension payloads; install/validate/remove output blocks; new deterministic error codes. |

Data flow (install): CLI/HTTP install → source fetch (unchanged) → installer root detection (tri-state triage) → hardening + scan (unchanged) → **per-`InstanceKey` lifecycle coordinator** (the extension manager's existing per-instance operation coordination, extended to own every mutating path — CLI, HTTP/UDS, marketplace, dev reload, native tools): stage → validate → move into `$COMPOZY_HOME/extensions/<name>` → registry transaction (`format`, diagnostics from the synthesis pass) → post-commit cleanup. Crash recovery at boot: a staged or final directory without a registry row is removed before the daemon serves extension traffic (registry is the authority); retries are idempotent. The data directory is **not** created at install — its path is derived at synthesis and the directory is created/verified immediately before the first stdio server launch (the frozen `_dx.md` contract), with creation failure reported as live per-server health, never as ingestion degradation. Data flow (load/enable/session): manifest load branch → `agentplugin.Load` → synthesis → kit publication (skills as explicit paths, MCP servers via `mcp_server` resources) → session merge — remote servers consumed daemon-side by the hosted-proxy executor; stdio servers forwarded to providers with `PLUGIN_ROOT`/`PLUGIN_DATA` baked into env at synthesis.

## Architectural Boundaries

- `internal/mcppolicy` is a pure **leaf** (stdlib only). It is the one home of header/URL policy — imported by `internal/config` (sidecar validation), `internal/extension/agentplugin` (ingestion), and `internal/mcp` (executor enforcement). No other package re-implements any part of that policy.
- `internal/extension/agentplugin` is a near-leaf with a **closed import list**: stdlib + `internal/frontmatter` + `internal/mcppolicy`. It must not import `internal/extension`, `internal/config`, `internal/skills`, or `internal/api/contract` — it exposes its own `Package`/`Diagnostic` types; converters live in the importing packages. It performs *standard-conformance* frontmatter checks only (agentskills rules: name==dir, description bounds) via `internal/frontmatter`; canonical skill parsing authority remains `internal/skills`, which parses the synthesized explicit paths downstream exactly as it does for native extensions today — one parse authority, no duplicate skill semantics.
- `internal/extension` imports `agentplugin` (synthesis + load branch). `internal/registry` imports `agentplugin` (root triage only). `internal/cli` and `internal/api/core` reach it exclusively through `internal/extension`.
- `internal/config` changes are self-contained (server struct + validation via `mcppolicy`); `internal/mcp` consumes `config.MCPServer` as today.
- `internal/daemon` remains the sole composition root; no new wiring outside it. No package imports `daemon/`, `api/`, or `cli/`.
- `magefiles/boundaries.go` gains the `mcppolicy` leaf rule and the `agentplugin` closed-import rule in the same commit that creates each package.

## Implementation Design

### Core Interfaces

```go
// internal/extension/agentplugin — closed-import conformance layer
// (allowed imports: stdlib + internal/frontmatter + internal/mcppolicy)
type SchemaStatus int

const (
	SchemaSupported          SchemaStatus = iota // exact 1.0.0 identifier
	SchemaUnsupportedVersion                     // agent-plugins.org prefix, other version
	SchemaUnrelated                              // anything else, incl. missing $schema
)

// ClassifyManifest reads dir/plugin.json (regular file only) and triages its $schema.
func ClassifyManifest(dir string) (status SchemaStatus, declared string, err error)

// ValidateName enforces the standard's name rule without lookahead (RE2-safe):
// length 1..64, byte-level charset, alnum edges, no "--", no "..".
func ValidateName(name string) error
```

```go
// Load runs the full fatality ladder. Fatal manifest problems return *ManifestError;
// component problems land in Package.Diagnostics and never fail the load.
type LoadOptions struct {
	DataDir string // absolute PLUGIN_DATA target used for ${PLUGIN_DATA} expansion
}

func Load(dir string, opts LoadOptions) (*Package, error)

type Package struct {
	Name, Version, Description   string
	Author                       *AuthorInfo // standard's author object; nil when absent
	Homepage, Repository, License string
	Keywords                     []string
	Skills                       []SkillRef   // immediate children of skills/ only
	Servers                      []ServerSpec // stdio + streamable-http survivors
	Diagnostics                  []Diagnostic // every skip, scoped
}

// AuthorInfo mirrors the standard's closed author object. All three fields are
// type-validated per the fatality ladder; the whole struct is retained for
// validate output — synthesis projects none of it (the extension manifest has
// no author field; marketplace author display comes from feed entries).
type AuthorInfo struct{ Name, Email, URL string }

type SkillRef struct{ Name, Dir, SkillFile string }

type ServerSpec struct {
	Name, Transport string // "stdio" | "streamable-http"
	// CWD normalization (standard §7.2.1/§9.2): omitted → the package root;
	// present values must be exactly "./…", "${PLUGIN_ROOT}"[/…], or
	// "${PLUGIN_DATA}"[/…]. Containment is validated against TWO canonical
	// domains — the package root and the data root — chosen by the prefix;
	// a ${PLUGIN_DATA}-rooted cwd is legal and resolves outside the package.
	Command, CWD string
	Args         []string
	Env          map[string]string // expanded; PLUGIN_ROOT/PLUGIN_DATA injected last
	URL          string            // streamable-http; policy-checked, never expanded
	Headers      map[string]string // streamable-http; policy-checked
}

type Diagnostic struct{ Scope, Message string } // "manifest" | "skills" | "skill:<dir>" | "mcp" | "mcp:<name>"

type ManifestError struct{ Issues []Issue }
type Issue struct{ Path, Message string }
```

```go
// internal/extension — synthesis + persistence additions
type ExtensionFormat string

const (
	FormatCompozy     ExtensionFormat = "compozy"
	FormatAgentPlugin ExtensionFormat = "agent-plugin"
)

// SynthesizeAgentPluginManifest converts a validated package into a resource-only
// manifest: explicit per-skill paths, MCP servers mapped onto MCPServerConfig,
// empty provides/permissions/subprocess, min_compozy_version waived (ADR-005).
func SynthesizeAgentPluginManifest(pkg *agentplugin.Package, rootDir string) (*Manifest, error)

// ExtensionInfo gains:
//   Format            ExtensionFormat            // persisted column
//   IngestDiagnostics []contract.DiagnosticItem  // persisted JSON, code extension_agent_plugin_component_skipped
```

```go
// internal/extension — manifest MCP declaration widening (extension.toml surface)
type MCPServerConfig struct {
	Command   string            `toml:"command,omitempty"        json:"command,omitempty"` // now optional: required iff transport is stdio
	Args      []string          `toml:"args,omitempty"           json:"args,omitempty"`
	Env       map[string]string `toml:"env,omitempty"            json:"env,omitempty"`
	SecretEnv map[string]string `toml:"secret_env,omitempty"     json:"secret_env,omitempty"`
	Transport string            `toml:"transport,omitempty"      json:"transport,omitempty"` // "" | "stdio" | "http"
	URL       string            `toml:"url,omitempty"            json:"url,omitempty"`
	Headers   map[string]string `toml:"headers,omitempty"        json:"headers,omitempty"`
}
```

```go
// internal/config — canonical session server widening (ADR-006)
type MCPServer struct {
	// … existing fields …
	Headers       map[string]string `json:"headers,omitempty"        yaml:"headers,omitempty"        toml:"headers,omitempty"`
	// SecretHeaders is DECLARATIVE: header name → secret reference (vault ref /
	// binding identity), mirroring SecretEnv. Resolved values NEVER enter this
	// field or any serializable type — the MCP executor resolves references
	// immediately before request construction into a local http.Header only.
	SecretHeaders map[string]string `json:"secret_headers,omitempty" yaml:"secret_headers,omitempty" toml:"secret_headers,omitempty"`
}
```

```go
// internal/mcppolicy — the single header/URL policy authority (new leaf)
type HeaderSource int

const (
	SourcePackageFixed   HeaderSource = iota // declared in package bytes (mcp.json)
	SourceOperatorSecret                     // operator binding (--remote-header)
	SourceOAuthOwned                         // derived by the OAuth broker
)

// ValidateHeaders enforces one deterministic outcome per violation:
//   - RFC 7230 token names; CR/LF-free values; case-insensitive duplicate
//     rejection across the UNION of fixed and secret maps.
//   - content-type and the mcp-* family: always rejected, every source.
//   - Sensitive set (authorization, proxy-authorization, cookie, set-cookie):
//     SourcePackageFixed → reject that server entry with a stable scoped code
//     (never stripped, never silently dropped);
//     SourceOperatorSecret → authorization allowed iff the server has no OAuth
//     auth configured; the rest of the sensitive set is always rejected.
func ValidateHeaders(fixed, secret map[string]string, source HeaderSource, authEnabled bool) error

// ValidateRemoteURL: absolute http/https, no userinfo, no fragment; plain
// http only for loopback hosts.
func ValidateRemoteURL(raw string) error
```

### Data Models

New columns (global stream, appended Goose migration + refreshed atlas/sqlc via `make codegen`):

| Column | Type | Purpose |
| --- | --- | --- |
| `extensions.format` | `TEXT NOT NULL DEFAULT 'compozy'` | Matchable format scalar (`compozy` \| `agent-plugin`) for the **global** instance; payload/list projection |
| `extensions.ingest_diagnostics_json` | `TEXT NOT NULL DEFAULT '[]'` | Recorded component skips (`[]DiagnosticItem`) for the global instance |
| `extension_dev_links.format` | `TEXT NOT NULL DEFAULT 'compozy'` | Format of the **workspace** dev-link instance — instance truth per `InstanceKey`, never inferred from a same-named global row |
| `extension_dev_links.ingest_diagnostics_json` | `TEXT NOT NULL DEFAULT '[]'` | Workspace instance's recorded skips; written atomically with `bundle_generation` on link/reload |
| `extension_env_bindings.mcp_server` | `TEXT NOT NULL DEFAULT ''` | Remote-header binding: target server name (empty = plain env binding) |
| `extension_env_bindings.header_name` | `TEXT NOT NULL DEFAULT ''` | Remote-header binding: header that carries the value the executor resolves at connection time |

Instance scoping rule: `format` + ingest diagnostics live on the row that owns the instance — `extensions` for published (global) installs, `extension_dev_links` for workspace dev links. Every projection resolves through `InstanceKey{Name, WorkspaceID}`; a global-native extension and a same-named workspace plugin dev link never share either value, and reload replaces the dev link's diagnostics in the same transaction that records its new generation.

Migration contract (L-008): all six columns land in **one** appended gap-free Goose migration on the global stream; the declarative schema, `atlas.sum`, and sqlc output refresh via `make codegen`; the binding both-or-neither invariant is enforced in storage — `CHECK ((mcp_server = '' AND header_name = '') OR (mcp_server <> '' AND header_name <> ''))`; applied-migration bytes stay immutable; the canonical fresh/reopen/ahead/integrity/equivalence suites extend to cover all six columns and existing-row backfills.

Side-table-vs-JSON decisions: `format` is a typed column (list/filter dimension — matchable state). Ingest diagnostics are a JSON column, not a side-table — they are opaque display data replayed verbatim to surfaces, never filtered or joined in SQL; a side-table would add schema for no query. The remote-header mapping is two typed columns on the existing binding row (matchable at session-start resolution; both empty = plain env binding, today's exact behavior — no dual write paths).

Other data facts: package data lives under a **dedicated `HomePaths`-owned root that cannot overlap managed installs** — `$COMPOZY_HOME/extension-data/<name>` for global installs and `$COMPOZY_HOME/extension-data/<name>@ws-<workspace_id>` for dev links. The old in-namespace idea (`extensions/data/…`) is rejected: `data` is a valid package name, and an extension named `data` would have made its install root the parent of every data directory, turning its update/remove into cross-instance data loss. Encoding is collision-free by construction: `@` cannot appear in package names (charset `a-z0-9.-`), and each encoded key is a single containment-checked segment. Directories are created only when the package declares at least one stdio server; preserved bit-for-bit on update; removed on `remove` (reported-but-non-fatal deletion failure). Catalog feed entries gain optional `"format"` as a **hard cut at the current feed `manifest_version`** — reader, in-repo feeds, fixtures, and projection update in one change; no version bump, no pre-bump compatibility behavior, no old-daemon degradation planning (zero production users; a stale local daemon refreshing the live feed simply upgrades). No new frontmatter fields, no new config keys.

### API Endpoints

No new routes (parity with `_dx.md`). Changes inside existing handlers — **one public projection, consistent across every surface**:

- `ExtensionPayload` gains `format` (always present) and `diagnostics` populated from the owning instance row (global or dev link), merged with live runtime MCP health. **One list contract**: `GET /api/extensions` items, `compozy extension list -o json`, and `compozy__extensions_list` all carry the full payload — `format` and `diagnostics` included (uniform payload builder, no list-only projection). Diagnostic ordering is total and repeatable: source set (recorded ingest → live runtime) → scope/server name → code → message. All new diagnostic codes register under the canonical category `extension`.
- `ExtensionInventoryPayload` gains the same `format` + `diagnostics` fields — inventory is the kit view and owns the skipped-components presentation (`_uiux.md` S4 reads it). CLI `inventory`, HTTP/UDS inventory, and `compozy__extensions_inventory` all carry them.
- The computed human `Summary` acknowledges degradation: `running (N components skipped)` when recorded skips exist — never "running and healthy" beside warn diagnostics. Human `list` columns stay unchanged (frozen surface decision); list structured output carries the full payload fields above.
- `PUT /api/extensions/:name/secrets` widens with the typed binding target: each binding entry gains optional `mcp_server` + `header_name` (both-or-neither — enforced at the storage boundary by a table CHECK constraint, so no partially-mapped binding can exist), the same domain operation the CLI verb calls — one primitive for manual and agent paths (L-004). OpenAPI + generated TS refresh via `make codegen`.
- `POST /api/extensions` maps the new failure kinds onto the existing error-kind enum: `extension_agent_plugin_client_layout`, `extension_agent_plugin_not_manifest`, `extension_agent_plugin_schema_unsupported` → 422 generic envelope; `extension_agent_plugin_manifest_invalid` → 422 validation envelope with `issues[]` built from `ManifestError`.
- Validate remains local (CLI + native tool) — no server route, matching today's surface; the CLI validate branch calls `agentplugin` directly through `internal/extension`.
- HTTP/UDS parity holds automatically (shared `BaseHandlers`); parity tests extend the existing transport-parity suite.

## Integration Points

Only pre-existing external touchpoints: source fetch (GitHub/git, unchanged), vault secret resolution for bindings (unchanged mechanism, new header target), and the daemon's outbound MCP connections — which now attach validated `Headers`/resolved `SecretHeaders` through the existing authorized `RoundTripper`. External-call timeout discipline is inherited from the executor client. The agent-plugins.org schemas are **not** fetched at runtime — schema identifiers are versioned constants (the standard mandates no-fetch validation).

## Impact Analysis

| Component | Impact Type | Description and Risk | Required Action |
| --- | --- | --- | --- |
| `internal/registry` installer | modified | Third root layout + triage; low risk (detection-only change, hardening untouched) | Extend `installer_metadata` detection + tests |
| `internal/extension` | modified + new subpackage | Load branch, synthesis, persistence columns, data-dir lifecycle; medium risk (core path) | New `agentplugin` package; migration; suite extensions |
| `internal/config` | modified | `MCPServer` widening (internal wire fields only); medium risk (session wire) | Shared `mcppolicy` validation; redaction tests; no sidecar/editor surface |
| `internal/mcp` executor | modified | Header attachment on existing seam; low risk | Redaction tests for secret headers |
| Secrets bindings | modified | Two columns + resolution path; low risk | Hygiene tests (values never in reads) |
| `internal/marketplace` + `catalog/` | modified | Optional entry field, hard cut at the current feed version; low risk | Reader + feeds + fixtures + projection in one change |
| CLI / API / native tools | modified | Output blocks, payload fields, error codes; low risk | Output + parity + tool descriptor tests |
| `web/` | modified | Badge variant, skipped rows, card badge; low risk | Vitest + Playwright per `_uiux.md` |
| `packages/site` + `skills/compozy` | modified | Install/manifest/secrets docs + interop page; official skill reference update | Docs tasks |

**Delete targets: none — additive feature.** No-fallback clauses: no heuristic ingestion of client-specific layouts (deterministic error only); no `sse` support path behind a flag; no compat alias for any widened field; the registry missing-manifest error message is updated in place to name all three roots (old copy replaced, not aliased).

## Extensibility Integration Plan

- **Extension manifests**: `MCPServerConfig` widening (transport/url/headers) is a public manifest-surface change — `manifest.mdx` + validation and encode/decode paths updated together; unknown-field strictness unchanged. The **workspace `mcp.json` sidecar, `compozy config` MCP verbs, and web MCP editor are explicitly unaffected** in this spec (round-2 B-005 cut): `Headers`/`SecretHeaders` are internal wire fields; exposing them on operator config surfaces is a future spec with its own frozen contract.
- **Installer/registry**: third root layout is the extension-surface change itself; source union, trust tiers, and provenance untouched.
- **Hooks, provides, permissions, bridge SDKs, Host API**: unaffected — synthesis emits empty `capabilities.provides`, empty `permissions.requires`, no subprocess, no hooks; checked surfaces: `capabilities.go`, `extensionprotocol` method set, bridge contract (no shape change anywhere).
- **Skills**: ingested skills flow through the existing kit publication path (`skill` resources, explicit paths); the skills loader itself is unchanged — agentskills-only frontmatter extras (`license`, `compatibility`, `allowed-tools`) surface as existing unknown-field log warnings, accepted (adapter already validated them).
- **MCP sidecars**: unaffected — the sidecar file format does not gain the header fields in this spec (see the manifests bullet above); the plugin's own `mcp.json` is read by `agentplugin`, never by the sidecar parser.
- **Registries/marketplace**: feed entry field as a hard cut at the current feed version.
- **Protocol docs / official skill**: `skills/compozy/references/extensions.md` gains the agent-plugin install path, format field, diagnostics semantics, and `--remote-header`.

## Agent Manageability Plan

Consistent with `_dx.md` (frozen): no new verbs or routes. Agents operate the feature through `compozy extension install|validate|status|inventory|list|update|remove|secrets` and `compozy__extensions_*` with `-o json|jsonl|toon` structured output; `format` and scoped `diagnostics` fields make degradation machine-detectable; the deterministic `extension_agent_plugin_*` codes are branchable without prose parsing; HTTP/UDS carry identical payloads (shared handlers + parity tests). Secret-binding operations (including `--remote-header`) are agent-operable through the structured CLI verb and the widened HTTP/UDS secrets contract — all three call the same domain operation with identical deterministic errors. **Deliberately no `compozy__extensions_secrets_*` native tools**: the existing tool set has zero secrets tools by design (secret material and vault refs stay out of model-visible tool traffic); the brokered CLI/HTTP/UDS path is the agent path, and this posture is unchanged, not an omission. Consent-requiring installs in non-interactive contexts fail deterministically (`--allow-unverified` without `--yes` → structured error), never hang.

## Config Lifecycle

**No `config.toml` keys added, changed, or removed.** Evidence: format detection is content-based (not policy — decided pre-spec); trust reuses `extensions.trust.allow_unverified` verbatim; checked surfaces — `ExtensionsConfig` (trust/sources/dev/resources), settings section `hooks-extensions`, marketplace config (base URL/TTL untouched). **No operator config artifact changes either** (round-2 B-005 cut): the workspace `mcp.json` sidecar format, `compozy config` MCP verbs, and the web MCP editor are unchanged — `Headers`/`SecretHeaders` are internal wire fields produced by synthesis and binding projection only, with redaction tests for every read surface. The one operator-facing config-adjacent change is the widened secrets binding contract (CLI/HTTP/UDS), which ships with docs + OpenAPI + tests in the same change.

## Testing Approach

Strategy (all concrete cases in [_tests.md](_tests.md)): table-driven Go units with `t.Run("Should …")` + `t.Parallel`, `-race`; fakes only at I/O boundaries (vault resolver, HTTP transport). The `agentplugin` package is pure → exhaustive unit coverage co-located in `internal/extension/agentplugin/`; canonical suites extended, never duplicated: `internal/extension` (synthesis/load/persistence), `internal/registry` (detection), `internal/config` (server + sidecar), `internal/daemon` (kit-resource integration + distribution E2E on the existing harness), CLI output suites, web Vitest + Playwright lanes. Fixture strategy imports the reference clients' adversarial patterns: a committed conformant testdata package (skills + all three transports + a bad skill + a bad server), self-referential placeholder roots, 8-bad-1-good isolation sets, real symlink escapes, real `git init` + `file://` installs — **no live network in any lane**; the distribution E2E uses a local archive/git fixture. Gates: scoped `make gate` per task, `make gate-full` at workstream close.

## Development Sequencing

### Build Order

1. `internal/extension/agentplugin` — reader/validator/expansion + full unit suite (pure; gate: package tests).
2. `internal/config` widening — `Headers`/`SecretHeaders` as internal wire fields + `internal/mcppolicy` shared policy (gate: config suites).
3. Registry detection — third root + triage (gate: installer suites).
4. Extension synthesis + persistence — load branch, migration (`format`, diagnostics, binding columns), min-version waiver, data-dir lifecycle (gate: extension + store suites, `make codegen-check`).
5. Executor headers + secrets `--remote-header` — resolution, redaction (gate: mcp + daemon secret hygiene suites).
6. CLI/API/native-tool projections — output blocks, payload fields, error codes, parity (gate: cli/api suites).
7. Web + catalog — badge, skipped rows, feed field hard cut (reader + feeds + fixtures + projection in one change; gate: bun lanes + Playwright).
8. Docs + official skill + QA flags — site pages, `skills/compozy`, `docs/qa/scenarios` additions; then the trailing tasks in evidence order: `qa-report`, `qa-execution` (whose scenarios include the provider matrix and the 8-item conformance walk), and **only then** the Phase 2 compatible-clients handoff. The external claim must cite the recorded QA evidence, disclose the OpenClaw ACP limit, and link to deployed docs (L-007, L-026).

Behavior-changing edits are isolated to steps 2–7; step 1 is pure addition; docs close the loop. Each step lands with its tests in the same change.

### Technical Dependencies

No **runtime** dependencies. One **one-time authoring** dependency exists: at step 1, both standard schemas are re-downloaded byte-for-byte to freeze the validator constants, and the pinned identifiers + content digests are recorded in the task evidence (HANDOFF staleness warning) — after that, nothing external is fetched at build, test, or run time. Internal ordering: step 4 depends on 1–3; 5 depends on 2; 6 depends on 4–5; step 7's feed edit ships with its reader in the same change (hard cut).

## Monitoring and Observability

Existing extension lifecycle events (`extension.lifecycle.loaded/failed`, install/update/remove event set) fire unchanged and gain the `format` correlation field; ingest skips are visible as the persisted diagnostics (no new durable event types — recorded state, not a stream). **Runtime MCP health has an explicit owner with full lifecycle semantics**: an in-memory runtime-health registry inside the MCP executor, keyed by `InstanceKey + bundle generation + server name`, written only at the call sites that observe an outcome (launch failure, auth failure, connection failure, successful exchange). Ordering is monotonic per key (an older failure can never overwrite a newer success — observations carry a per-key sequence); success **clears** the entry; reload/update/disable/remove **evict** every entry of the affected instance+generation; daemon restart starts empty (unknown, not restored). Entries project as diagnostics with code `extension_mcp_server_unhealthy`, canonical category `extension`, `data_freshness: "live"`, redacted per the secret rules, and merge after the immutable ingest set on status/inventory/list across CLI/HTTP/UDS/native surfaces — package validity (ingest-time) and runtime availability (live) are never conflated, and one unhealthy server never marks siblings. Structured logs at synthesis carry `extension_name`, `format`, `diagnostic_count`, and per-skip `scope`. No metrics/alerting additions — install-time operations, not hot paths.

## Technical Considerations

### Key Decisions

- **E2E fixture**: a committed conformant testdata package + local git/archive distribution fixture — no dependency on third-party repos in CI; the HANDOFF's "real public plugin" idea is satisfied by mirroring the official example's shape into testdata (AWS kits are not root-layout conformant and would be a false fixture).
- **`name == directory` for skills** is enforced adapter-side (the skills format's rule; skip + report on mismatch) — deliberately not added to the native skills loader, whose collision semantics are precedence-based.
- **Deterministic synthesis** is a hard requirement, with its inputs named: identical package bytes **plus** identical canonical inputs (package root path, data-directory path, instance key) → identical manifest (ordering fixed by sorted directory walks). The content checksum covers package bytes only and is independent of the synthesized absolute paths — two instances of the same content share a checksum while their expanded env differs.
- **Update re-runs the full ladder**; diagnostics replace the persisted set atomically with the content move.
- **Placeholder expansion at synthesis** (not launch): stdio servers reach providers with concrete absolute paths in env — satisfying conformance item 6 on both the hosted-proxy and degraded delivery paths.

### Known Risks

- **Standard is frozen but untagged** (no git tag/release): validator pins the exact schema identifiers; any future version is a new deliberate mapping. Likelihood of near-term change: low (spec text untouched since 2026-07-24).
- **Curated entries drifting non-conformant upstream**: backstopped by the install-time deterministic layout error (ADR-002 risk, accepted).
- **Feed field hard cut vs stale local daemons**: a daemon older than the reader change fails to decode plugin-format entries until upgraded — accepted alpha cost (zero production users, L-006); no compatibility machinery exists to maintain.

## Safety Invariants

1. Every path read from a package resolves inside its containment domain after symlink resolution — package-content paths inside the package root, data-rooted working directories inside that instance's data root (two canonical domains, selected by the declared prefix); escapes from either domain are refused at the ladder's narrowest boundary.
2. Placeholder expansion is single-pass and applies only to `args` elements, `env` values, and `cwd` — never `command`, `url`, or `headers`.
3. `PLUGIN_ROOT`/`PLUGIN_DATA` cannot be overridden by package-declared env; the daemon sets them last.
4. Host environment variables are never interpolated into adapted package configuration; unrecognized placeholder text stays literal.
5. Package-declared header/env values are non-secret package data under a **source-aware policy** (`internal/mcppolicy`): package-declared sensitive headers (`authorization`, `proxy-authorization`, `cookie`, `set-cookie`) reject that server entry with a stable scoped code — never stripped, never silently dropped; operator bindings are the only credential path.
6. Secret references (never values) live in declarative config; the executor resolves them immediately before request construction into a non-serializable request header only. Resolved values never appear in any serializable type, log, status/API/UI read (key names only), clone, or provider projection — and remote servers never serialize toward ACP providers at all.
7. One policy authority (`internal/mcppolicy`) serves sidecar validation, plugin ingestion, and the executor: `content-type` and `mcp-*` always rejected; `authorization` via operator binding allowed only when the server has no OAuth auth configured; duplicate detection is case-insensitive across the union of fixed and secret header maps. No second implementation exists anywhere.
8. A synthesized instance carries zero Host API permissions, zero provides, and no extension subprocess — resource-only by construction, under every trust tier's ceiling.
9. Root-layout precedence is a fixed matrix decided before validation: (a) `extension.toml` + `SKILL.md` both at root → the existing hard error, unchanged, regardless of `plugin.json`; (b) exactly one native root present → it wins over `plugin.json`; (c) no native root + conformant `plugin.json` → portable ingestion; (d) client-specific directories never participate in selection; (e) a selected invalid root fails the install with no fallback to another format.
10. Every mutating lifecycle path (CLI, HTTP/UDS, marketplace, dev reload, native tools) routes through the per-`InstanceKey` lifecycle coordinator — one authoritative primitive, no parallel path. Commit order is fixed: stage → validate → final move → registry transaction → post-commit cleanup; a crash before the registry commit leaves artifacts the boot reconciliation removes (registry is the authority); retries are idempotent. Interrupted operations expose either the previous state or the completed state, never a hybrid; failed installs register nothing.
11. The package data directory is preserved bit-for-bit across updates and removed on `remove`. When deletion fails, the residue is atomically quarantined under a non-reusable identity (renamed out of the instance's deterministic key) so a subsequent same-name install always starts clean; if quarantine also fails, the removal itself fails deterministically — a completed remove never leaves reachable data under the instance's key.
12. Fatal manifest errors reject the whole package; component-level failures skip exactly that component, are recorded as scoped diagnostics, and never suppress sibling components.

## Assumptions and Defaults

- Supported standard version: exactly `1.0.0` (both schema identifiers pinned as constants; no fetch, no ranges).
- `format` values: `compozy` | `agent-plugin`; default column value `compozy` backfills existing rows truthfully.
- Adapted manifests carry no `min_compozy_version` gate (waived, ADR-005); native manifests keep the gate unchanged.
- Data directory default: `$COMPOZY_HOME/extension-data/<name>` (global) / `$COMPOZY_HOME/extension-data/<name>@ws-<workspace_id>` (dev links) — a dedicated `HomePaths`-owned root, disjoint from `extensions/` by construction. The path is derived at synthesis; the directory is created and verified immediately before the first stdio server launch (the frozen `_dx.md` contract) — never at install, update, or enable. Creation failure is live per-server health, not ingestion degradation.
- Legacy `sse` entries: shape-validated, then skipped (`sse transport is not supported`); malformed ones report `invalid mcp server entry`.
- `extensions` namespaces and client-owned namespace directories: ignored without content validation (standard-mandated), noted factually in diagnostics only when the value is non-object.
- Empty packages (manifest only) install as valid zero-resource instances.
- Dev links of portable packages compute the generation identity as the source-tree checksum (no build step exists for them).
- Update source-of-truth: content checksum (existing flow); manifest `version` is display metadata.
- Web feed marker is display metadata; install-time detection is always authoritative.

## Architecture Decision Records

- [ADR-001: Ingest as a third install layout of the existing extension system](adrs/adr-001.md) — one concept, resource-only synthesis, format as interchange.
- [ADR-002: Marketplace via existing curated feeds](adrs/adr-002.md) — feed entries + badge, no dedicated section.
- [ADR-003: Conformance diagnostics are first-class product output](adrs/adr-003.md) — full ladder visible on install/validate/status/inventory.
- [ADR-004: Verbatim package-name identity](adrs/adr-004.md) — dots preserved everywhere, no normalization.
- [ADR-005: In-memory manifest synthesis](adrs/adr-005.md) — `plugin.json` single source of truth; min-version gate waived.
- [ADR-006: Fixed MCP headers on the canonical server config](adrs/adr-006.md) — daemon-side consumption, `Headers`/`SecretHeaders` split, header policy.

## Compozy Impact Audit

- **Native tools**: no new tool IDs; `compozy__extensions_install/_update/_validate/_info/_list/_inventory` outputs gain `format` + `diagnostics` fields (descriptor output schemas + digests refresh); risk flags unchanged (`_install` mutating, `_remove` destructive); availability diagnostics unchanged. Checked: `builtin_extension_ids.go` set — no additions.
- **Extensibility and hooks**: installer third layout + manifest `MCPServerConfig` widening are the extension-surface changes; hooks/provides/permissions/bridge SDKs/Host API method set untouched (checked: `capabilities.go`, `host_api_methods_gen.go`, bridge contract). The workspace MCP sidecar stays closed to header fields; Agent Plugins `mcp.json` is handled only by the ingestion adapter. Official skill updated.
- **Workspace data isolation**: identical to managed extensions — published installs are global instances, dev links workspace instances keyed by `InstanceKey`; every new datum is instance-scoped on the row that owns the instance (`format` + ingest diagnostics on both `extensions` and `extension_dev_links`, data dirs keyed by the instance encoding, binding columns workspace-scoped as today); a global-native extension and a same-named workspace plugin dev link never share state; no new cross-workspace read paths (checked: payload projection through `InstanceKey`, SSE extension events, kit publication scopes, web cache reconciliation).
- **Official Compozy skill**: `skills/compozy/references/extensions.md` updated — agent-plugin install path, format field, diagnostics semantics, `--remote-header`, data-dir lifecycle.
- **Config lifecycle**: zero `config.toml` changes and zero operator config-artifact changes (evidence in Config Lifecycle section — sidecar/config verbs/web MCP editor explicitly unaffected); the secrets HTTP/UDS binding-target contract ships with docs + validation + OpenAPI/codegen + tests in the same change.
- **Web/Docs impact**: `web/` — badge variant, skipped-components section, marketplace card/detail badge (MCP editor untouched — B-005 cut); `packages/site` — `extensions/install.mdx` (third layout + source table rows), `extensions/manifest.mdx` (MCP widening), `extensions/secrets.mdx` (`--remote-header`), new short interop page in the extensions section, CLI generated pages refresh.
