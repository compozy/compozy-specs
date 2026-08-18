# TechSpec: Extension DX Overhaul (ext-improvs)

Companion test contract: `_tests.md`. Requirements source: `_brief.md` (no PRD exists for this program; the brief is the requirements document — R1–R11 referenced throughout). ADRs: `adrs/adr-001..008.md`.

## Executive Summary

This program makes the extension system usable end to end by a developer (or an agent) outside this repository, and opens distribution without a gatekeeper. Four architectural cuts carry the design: **code-first authoring** — the SDK definition is the single source of truth and `compozy extension build` generates the manifest, deleting the dual-declaration/digest-drift class (ADR-001); a **first-class dev lane** — `compozy extension dev` links the author's directory with zero marketplace trust ceremony, single-extension reload, and watch (ADR-002); **uniform generated SDK contracts** — `compozy-codegen` emits the Go contract layer from the same daemon specs that already feed TypeScript, both SDKs get published, and drift becomes a CI failure (ADR-003); and **GitHub/git-first distribution** — install sources become a union (`curated|github|git|local_path`; dev is a lifecycle overlay, not a source), `compozy extension publish` creates integrity-checked releases (digest sidecars never elevate trust), and search fans out across sources (ADR-005). The authored permission surface collapses to one closed-set-validated list with derived consent areas (ADR-004), the extension config surface is consolidated under `[extensions.trust]`/`[extensions.sources]` (ADR-007), and extensions gain **contributed commands** — operator verbs declared as presentation metadata on a tool and executed through the existing tool runtime via `compozy extension exec` (ADR-008). The command surface ships fixture-proven with **no product command in this program**: the first product command — the legacy `compozy archive` port in dev-cycle, which additionally needs a task-domain closure primitive (peer-review R2 B-007) — is owned by the follow-up `cmd-archive` program (`.compozy/tasks/cmd-archive/_brief.md`).

The primary trade-offs: a build step becomes mandatory for subprocess extensions (hidden inside `dev`/`publish`); the manifest schema, install request contract, and config keys break hard (greenfield alpha — delete targets enumerated, no compat shims); and `sdk/go` becomes a nested Go module with its own release tag.

**MVP boundary:** all seven build phases (A–G) below are in scope for this program's `_tasks.md`, including docs, official-skill authoring content, and the QA pair. Post-MVP, explicitly out of this program: the external bridge SDK program (positioned by ADR-006, follow-up TechSpec), the three remaining Future surfaces in Linear AGH-105 (ACP providers / notification channels / web UI contribution — CLI-verb contribution is **in scope** via ADR-008), the workflow-archive program — the task-domain closure primitive plus the dev-cycle `archive` command — spun out to `.compozy/tasks/cmd-archive/_brief.md` (no product command ships on the ADR-008 surface here), executable command groups + group-level persistent flags + command depth beyond 2 + top-level fallback dispatch (all additive on ADR-008), npm-`create` wrapper for the scaffolder, self-hostable third-party registries, protocol-version negotiation (strict `"1"` stays; revisit at GA), and skills/ClawHub pipeline convergence.

## System Architecture

### Component Overview

- **`internal/extension` (modified, heavy)** — manifest v2 (permissions), build/validate (`build.go`), dev lane (`dev.go`: link records, reload), publish (`publish.go`), per-instance log ring buffer, trust gate simplification, provenance for `github|git` sources. Consumes `internal/extension/contract` describe types.
- **`internal/extension/contract` (modified)** — the **single generated-contract authority** (peer-review R2 B-003): `DescribePayload` (build-time contract printed by SDKs in describe mode), command/group specs (ADR-008), the existing `HostAPIMethodSpecs` + `BuildHookContracts`, plus the new consent-area derivation table. It composes runtime protocol types from `internal/extensionprotocol` (untouched base package). `internal/extensioncontract` never existed — that name leaked from the `sdkts` import alias (`internal/codegen/sdkts/generate.go:12`).
- **`internal/codegen/sdkgo` (new)** — Go contract generator mirroring `sdkts`; `cmd/compozy-codegen` gains `sdk-contracts-go` target wired into `codegen`/`codegen-check`.
- **`sdk/go` (restructured)** — nested module `github.com/compozy/compozy/sdk/go` (own `go.mod`); layers: transport (existing), `contracts/` (generated), ergonomics (`NewExtension`, `Tool[T]`, hooks, describe mode). Public conformance harness `sdk/go/extensiontest` covering all five provide capabilities.
- **`sdk/typescript` (modified)** — describe mode, generated required-methods map replaces the hand map in `capabilities.ts`, `trusted_workspace` + `invocation_id` surfaced to handlers, published to npm (public, licensed).
- **`internal/registry` (modified)** — `gitsrc/` source (new), GitHub release digest-sidecar verification, publish upload path in `github/`.
- **`internal/cli` (modified)** — new verbs `init|build|validate|dev|reload|logs|publish|commands|exec`; search union; human-mode diagnostic rendering; success output with next-step hints; jsonl parity; update column; two-phase flag projection for contributed commands.
- **`internal/daemon` + `internal/api/*` (modified)** — routes for dev/reload/logs/search/commands, install request union, list payload additions, native tools (9 new), doctor extension probe.
- **`internal/hooks` (modified)** — `HookSourceExtension` with its own priority tier.
- **`internal/config` (modified)** — ADR-007 sections; key registry; validation.
- **`web/src/systems/{marketplace,extensions}` (modified)** — update affordances, dev badge, local-path install, failure counters.
- **`packages/site` + `skills/compozy` + `catalog` (modified)** — rewritten authoring docs, quickstart, publish guide, official-skill authoring reference.

Data flow (authoring): author code → `compozy extension build` → describe-mode subprocess → `DescribePayload` → generated `dist/extension.toml` + bundle → (`dev` link | `publish` release | `install`) → daemon `Registry` + `Manager` → subprocess runtime.

## Architectural Boundaries

- `internal/extension` may import `internal/extensionprotocol`, `internal/subprocess`, `internal/hooks`, `internal/tools`, `internal/registry`, `internal/resources`, `internal/config`, `internal/diagnostics`, `internal/fileutil`, `internal/filesnap`. It must NOT import `internal/daemon`, `internal/api/*`, `internal/cli`, or `internal/marketplace` (marketplace search union composes in the daemon service layer).
- **New packages**: `internal/codegen/sdkgo` (imports `internal/extension/contract` sources only — the same package `sdkts` already imports aliased `extensioncontract` — and mirrors `sdkts`); `internal/registry/gitsrc` (implements `registry.Source`; imports `internal/registry` types only).
- `sdk/go` (nested module) imports **nothing** from `internal/*` — enforced by the existing `TestSDKHasNoDaemonInternalImports` plus a new boundaries rule; `sdk/go/contracts` is generated, never handwritten.
- Composition stays in `internal/daemon` (SD-008): dev-lane wiring, native tools, search union, doctor probe registration all land in existing `boot_extension*`/`extension_manager_wiring.go` files.
- `magefiles/boundaries.go` is updated in the same commits that introduce `internal/codegen/sdkgo` and `internal/registry/gitsrc`.
- CLI-side watch loop lives in `internal/cli` (uses `internal/filesnap`); the daemon never watches author directories.

## Implementation Design

### Core Interfaces

Describe contract (printed by SDKs on `__describe` argv; consumed by build). The authoritative schema lives in `internal/extension/contract` — the source the TS generator already reads (`internal/codegen/sdkts/generate.go:12`, aliased `extensioncontract`) — so daemon, Go SDK, and TS SDK representations plus conformance fixtures are all generated and byte-checked by `make codegen-check`; the authoring handshake is never a handwritten cross-package mirror (peer-review B-007, R2 B-003):

```go
// internal/extension/contract/describe.go — authoritative; generated into sdk/go/contracts + sdk/typescript generated
type DescribePayload struct {
    Name             string                                    `json:"name"`
    Version          string                                    `json:"version"`
    Description      string                                    `json:"description,omitempty"`
    Provides         []string                                  `json:"provides"`
    Permissions      []string                                  `json:"permissions"` // Host API method paths
    RequiresEnv      []string                                  `json:"requires_env,omitempty"`
    Subprocess       DescribeSubprocess                        `json:"subprocess"`
    Tools            []toolspkg.ExtensionToolRuntimeDescriptor `json:"tools,omitempty"`
    HookEvents       []string                                  `json:"hook_events,omitempty"`
    WatchSourceKinds []string                                  `json:"watch_source_kinds,omitempty"`
    CommandGroups    []ExtensionCommandGroupSpec               `json:"command_groups,omitempty"` // declared presentation groups (ADR-008, R2 B-005)
    SDK              DescribeSDKInfo                           `json:"sdk"`
}
type DescribeSDKInfo struct {
    Name              string `json:"name"`
    Version           string `json:"version"`
    ProtocolVersion   string `json:"protocol_version"`
    MinCompozyVersion string `json:"min_compozy_version"` // SDK-carried compat floor, stamped into the manifest
}
```

Build and validate (daemon-free; used by CLI `build|validate|dev|publish`):

```go
// internal/extension/build.go — every build lands in an immutable generation directory (peer-review B-006)
type BuildRequest struct {
    SourceDir string
    OutputDir string        // default <SourceDir>/dist — the ONLY dev-linkable root: the daemon resolves generation handles
                            // exclusively under <origin>/dist/gen-<hash>, so a custom OutputDir is for standalone packaging
                            // (e.g. CI publish) and can never back a dev link (R3 N-001); builds never mutate a prior generation
    BuildCmd  []string      // override; empty = detect: package.json "build" script → bun|npm run build; go.mod → go build
    Timeout   time.Duration // describe-mode ceiling; default 60s
}
type BuildResult struct {
    GenerationDir  string // immutable, fully validated before being returned
    GenerationHash string // content hash naming dist/gen-<hash>; the only generation identity that crosses transports (R2 B-002)
    ManifestPath   string
    Manifest       *Manifest
}
func BuildBundle(ctx context.Context, req BuildRequest) (*BuildResult, error)

type IssueSeverity string // closed enum, generated into OpenAPI + both SDKs (peer-review N-005)
const (
    IssueSeverityError   IssueSeverity = "error"
    IssueSeverityWarning IssueSeverity = "warning"
)
type ValidationIssue struct{ Path string; Line, Column int; Field, Message string; Severity IssueSeverity }
func ValidateBundle(dir string) (*Manifest, []ValidationIssue, error) // closed sets, TOML/JSON position on decode errors, derived consent summary
```

Dev lane and reload (peer-review B-002/B-003/B-006 — durable overlay, workspace binding, atomic generations):

```go
// internal/extension/dev.go — the published `extensions` row is never displaced; the overlay is a side table
type DevLink struct {
    ExtensionName    string
    WorkspaceID      string    // authorization boundary: agent surfaces bind this from trusted session scope
    OriginPath       string    // canonicalized (EvalSymlinks + containment) under the workspace root
    BundleGeneration string    // immutable generation dir currently active
    LinkedAt         time.Time
}
func (r *Registry) LinkDev(link DevLink, manifest *Manifest) error // upsert by (extension_name, workspace_id); no 409
func (r *Registry) UnlinkDev(name, workspaceID string) error       // side-table delete; published row resumes by resolution
func (r *Registry) ResolveActive(name, workspaceID string) (*ExtensionInfo, *DevLink, error) // dev link wins while present

// internal/extension/instance.go — the runtime identity every dev/lifecycle surface is keyed by (R2 B-001):
// subprocess, operation coordinator, last-good generation, log ring, status, events. A workspace with an
// active dev link resolves to (name, workspace_id); everything else resolves to the one global published
// instance (name, ""). Agent callers reach only instances their workspace resolves to.
type InstanceKey struct {
    Name        string
    WorkspaceID string // "" = the global published instance
}

// internal/extension/manager_lifecycle.go — one operation coordinator per *instance* serializes
// build/stage/swap/reload/watch; a failed activation restarts the last-good generation (never a torn state).
// Reload requires an active dev link for the key's workspace (published instances cycle only through
// install/update/enable/disable). generationHash selects a build output under the link's canonicalized
// origin — never a caller-supplied path (R2 B-002).
func (m *Manager) ReloadExtension(ctx context.Context, key InstanceKey, generationHash string) error
```

Manifest v2 (generated for subprocess extensions; handwritten only for resource-only):

```go
// internal/extension/manifest.go — Actions/Security deleted, Permissions added
type Manifest struct {
    Name, Version, Description string
    MinCompozyVersion string
    RequiresEnv  []string
    Resources    ResourcesConfig
    Capabilities CapabilitiesConfig // provides — unchanged
    Permissions  PermissionsConfig  // NEW
    Subprocess   SubprocessConfig
    Bridge       BridgeConfig
}
type PermissionsConfig struct{ Requires []string `toml:"requires,omitempty" json:"requires,omitempty"` }

// internal/extension/consent.go — table generated from internal/extension/contract
type ConsentArea struct{ Area, Access string } // e.g. {"sessions","read"}
func DeriveConsentAreas(methods []string) []ConsentArea
```

Generated Go contracts (new codegen target; excerpt of emitted shape):

```go
// sdk/go/contracts (GENERATED — compozy-codegen sdk-contracts-go)
type HostAPIMethod string
const HostMethodSessionsList HostAPIMethod = "sessions/list" // … all 87, typed params/result structs per method
func RequiredMethods(provide string) []HostAPIMethod          // single source; replaces hand maps in BOTH SDKs
type HookEvent string                                         // all 90 events + typed payload/patch structs
type ExtensionToolCallRequest struct {                        // full mirror incl. previously dropped fields
    ToolID, Handler, SessionID, InvocationID string
    TrustedWorkspace *ToolWorkspaceScope
    Input json.RawMessage
}
```

Distribution:

```go
// internal/registry/gitsrc/client.go
func NewClient(opts ...Option) *Client // implements registry.Source: shallow clone at ref → archive; Search returns ErrNotSupported

// internal/extension/publish.go — no credential field on the request (peer-review B-005): the GitHub
// credential is resolved server-side from an approved binding (daemon env GITHUB_TOKEN or
// vault:github/publish) and injected into the uploader; it never appears in tool inputs, request
// payloads, errors, events, or transcripts. The CLI-local verb resolves the env itself and never logs it.
type PublishRequest struct{ GenerationDir, Repository, TagName string; Draft bool }
type PublishResult struct{ ReleaseURL, AssetURL, DigestSHA256 string }
func PublishRelease(ctx context.Context, uploader githubreg.ReleaseUploader, req PublishRequest) (*PublishResult, error)
```

Logs and CLI error rendering:

```go
// internal/extension — per-instance bounded ring buffer (256 KiB), fed by subprocess stderr continuously;
// agent callers read only instances resolved for their workspace — the global instance's logs are
// operator-transport-only (R2 B-001)
type LogLine struct{ At time.Time; Stream, Text string }
func (m *Manager) Logs(key InstanceKey, limit int) ([]LogLine, error)

// internal/cli/root.go — human mode stops discarding diagnostics
func renderHumanExecutionError(err error) (string, bool) // "error: <msg>\n  <detail>\n  try: <suggested command>"
```

Hook source:

```go
// internal/hooks/types.go — new value; ordering.go gains the tier
const HookSourceExtension HookSource = 4 // name "extension"; DefaultHookPriority → 300 (Config 500 > Extension 300 > AgentDefinition 100)
```

Extension-contributed commands — presentation metadata on an extension tool, executed through the existing tool runtime (ADR-008):

```go
// internal/extension/contract/command.go — authoritative; generated into both SDKs like the describe schema
type ExtensionCommandSpec struct {
    Verb    string            `json:"verb"`              // path, "/"-joined, max depth 2: "archive" | "review/fetch"
    Summary string            `json:"summary"`
    Example string            `json:"example,omitempty"`
    Flags   map[string]string `json:"flags,omitempty"`   // CLI flag name → top-level input-schema field
}
type ExtensionCommandGroupSpec struct {
    Path    string `json:"path"`              // "review"; presentation node only — never executable
    Summary string `json:"summary"`
}

// internal/extension/command.go — read-model projection over active tool descriptors (no new storage)
type CommandDescriptor struct {
    Extension, Verb, ToolID, Summary, Example string
    Flags            []CommandFlag // projected from Spec.Flags × input schema (closed subset, ADR-008); canonical lexicographic order by flag name (R2 N-004)
    RiskClass        toolspkg.RiskClass
    ApprovalRequired bool // static approval metadata mirrored from the tool descriptor; invoke-time policy stays authoritative (R2 N-004)
}

// internal/extension/contract/command.go — the flag read-model shape is itself generated contract data
// (OpenAPI + both SDKs + the CLI's two-phase parser all consume this one struct — R3 B-004)
type CommandFlagType string // closed enum: the projected scalar after $ref resolution
const (
    CommandFlagString  CommandFlagType = "string"
    CommandFlagBoolean CommandFlagType = "boolean"
    CommandFlagInteger CommandFlagType = "integer"
    CommandFlagNumber  CommandFlagType = "number"
)
type CommandFlag struct {
    Name       string          `json:"name"`              // CLI flag name; the lexicographic sort key
    Field      string          `json:"field"`             // top-level input-schema property it projects
    Type       CommandFlagType `json:"type"`              // item type when Repeatable (array-of-scalar)
    Repeatable bool            `json:"repeatable"`        // repeated flag appends to the array field
    Required   bool            `json:"required"`          // schema-required; checked client-side before invoke
    Nullable   bool            `json:"nullable"`          // [T,"null"] scalar; absent flag omits the field (explicit null needs --input)
    Enum       []string        `json:"enum,omitempty"`    // canonical string forms; rendered in help, value-validated at invoke
    Default    json.RawMessage `json:"default,omitempty"` // schema default; help display only — the CLI never injects it
    Minimum    *float64        `json:"minimum,omitempty"` // numeric bounds; rendered in help, enforced at invoke
    Maximum    *float64        `json:"maximum,omitempty"`
}
func (m *Manager) Commands(workspaceID string) ([]CommandDescriptor, []ExtensionCommandGroupSpec, error)

// internal/cli — two-phase parse: host flags, then descriptor-projected flags
func buildCommandInput(desc CommandDescriptor, flags *pflag.FlagSet, rawInput string) (json.RawMessage, error)
```

### Data Models

Extension registry rows (global catalog stream, `compozy.db`; Goose migration per `eng-schema-migration` — hard cut, alpha reinstall accepted, enumerated in Delete Targets):

| Column | Type | Rationale (purpose + shape) |
| --- | --- | --- |
| `extensions.source` | `TEXT NOT NULL` | Existing column; value set unchanged (`dev` is **not** a source — peer-review N-001). The list projection derives an effective `dev (overrides published)` label when an active dev link exists. |
| `extensions.provides_json` | `TEXT NOT NULL DEFAULT '[]'` | `capabilities.provides` list. JSON column: opaque validated-in-Go list, never SQL-filtered. |
| `extensions.permissions_json` | `TEXT NOT NULL DEFAULT '[]'` | `Manifest.Permissions.Requires` (Host API method paths). JSON column, same reasoning; replaces the `actions`/`security` persistence, which is **deleted** in the migration. |

New side table — the dev overlay (peer-review B-002/B-003); the published `extensions` row is never displaced, so unlink-restores is a delete, not a data recovery:

| Column | Type | Rationale (purpose + shape) |
| --- | --- | --- |
| `extension_dev_links.extension_name` | `TEXT NOT NULL` | Overlay key; unique with `workspace_id`. |
| `extension_dev_links.workspace_id` | `TEXT NOT NULL` | Authorization boundary (L-033): agent dev/reload/logs bind this from trusted session scope; list/status/log/SSE projections filter by it for agent callers. |
| `extension_dev_links.origin_path` | `TEXT NOT NULL` | Canonicalized author source dir; must resolve inside the workspace root (invariant 2). |
| `extension_dev_links.bundle_generation` | `TEXT NOT NULL` | Immutable generation dir currently active; swapped atomically on reload (B-006). |
| `extension_dev_links.linked_at` | `TIMESTAMP NOT NULL` | Display + staleness diagnostics. |

**Side-table-vs-JSON decisions:** the dev overlay is a **typed side table** (`extension_dev_links`) because it is matchable state — resolved per (extension, workspace) on every activation, filtered by workspace in projections, and joined for the `overrides_published` flag; a JSON bag on the extensions row would recreate the L-012-forbidden pattern. `permissions_json` and `provides_json` are JSON columns — opaque validated-in-Go lists, never filtered via SQL. Provenance stays the existing opaque JSON blob (display-only). `update_available` is **not stored** — it remains a read-model projection joining installed rows against `marketplace_catalog_entries`/registry lookups at list time.

Contract payload additions (`internal/api/contract`, co-shipped with OpenAPI + generated TS + E2E mocks per L-007):

```
ExtensionPayload    += update_available bool, remote_version string, origin_path string,
                       consecutive_failures int, restart_backoff_ms int64, overrides_published bool,
                       digest_matched bool           // integrity fact; NEVER implies checksum_verified (B-001)
InstallExtensionRequest (hard cut) = { source: "curated"|"github"|"git"|"local_path",   // no "dev" member (N-001)
                       ref: string, version?: string, asset?: string, allow_unverified?: bool }
DevLinkRequest       = { origin_path: string, generation_hash: string }
                       // generation identity is a content-hash handle, never a path: the daemon canonicalizes
                       // origin_path (invariant 3), reconstructs <origin>/dist/gen-<hash> itself, and re-verifies
                       // the generation's manifest + digest before activation (R2 B-002)
                       // workspace_id is NEVER client-supplied: HTTP/UDS/native handlers bind it from
                       // the authenticated caller scope (operator CLI: resolved workspace; agent: session) (B-003)
ReloadExtensionRequest = { generation_hash: string }   // same handle contract as DevLinkRequest
ExtensionValidatePayload = { manifest?: ExtensionManifestSummary, issues: ValidationIssue[], consent_areas: ConsentArea[] }
ExtensionLogsPayload = { lines: { at, stream, text }[] }   // redacted at ingestion (B-005), workspace-filtered for agents
```

### API Endpoints

| Method | Path | Description |
| --- | --- | --- |
| POST | `/api/extensions/dev` | Link a built dev bundle (privileged; body `DevLinkRequest`; 200 `ExtensionPayload`) |
| POST | `/api/extensions/{name}/reload` | Rebuilt-bundle reload of one extension (privileged; 200 `ExtensionPayload`; 404/409) |
| GET | `/api/extensions/{name}/logs` | Ring-buffer lines; `?limit=`; `?follow=1` upgrades to SSE using the **named event `extension_log`** (clients register via `addEventListener("extension_log", …)` per L-017 — never the unnamed `message` event) |
| GET | `/api/extensions/search` | Source-union search: `?q=&sources=curated,github&limit=&cursor=` |
| GET | `/api/extensions/commands` | Contributed-command read model (ADR-008): `?extension=` filter; returns leaves (verb path, tool id, summary, projected `CommandFlag[]`, risk class, `approval_required`) + declared groups; workspace-filtered for agent callers |
| POST | `/api/extensions` | Install — request hard-cut to the source-union shape above |
| GET | `/api/extensions` | List — payload gains update/remote-version/failure fields |

All routes registered in both `internal/api/httpapi` and `internal/api/udsapi` (full parity, `internal/api/core` shared handlers). Dev, reload, and logs handlers bind `workspace_id` server-side from the authenticated caller scope (operator transport: the resolved workspace; agent native tools: trusted session scope via the canonical dispatcher) — it is never accepted from the request body, and agent-facing list/status/logs/SSE/event projections filter by that identity (peer-review B-003). Reload and logs act on the extension **instance** the bound workspace resolves to — `(name, workspace_id)` when a dev link exists, the global `(name, "")` instance otherwise; reload additionally requires an active dev link for that workspace, `POST /api/extensions/{name}/reload` takes `ReloadExtensionRequest`, and the global instance's logs are operator-transport-only (R2 B-001). `ExtensionStatusCode` gains: dev origin missing → 409 `ErrExtensionDevOriginMissing`; reload without an active dev link for the bound workspace → 409 `ErrExtensionNotDevLinked`; unknown/malformed/stale `generation_hash` → 400 `ErrExtensionGenerationInvalid`; validation-failed install → 400 with `issues` payload; `bridge.adapter` on an installed manifest → 400 deterministic "external bridge authoring is a planned follow-up" (ADR-006). `compozy extension validate|build|init|publish` are daemon-free CLI verbs (no routes); `search` requires the daemon.

## Integration Points

- **GitHub API** (`internal/registry/github`, existing client: explicit timeout, retries, `GITHUB_TOKEN` optional) — release resolution + asset download (existing), digest-sidecar fetch (new: `<asset>.sha256`, **integrity-only**: a match records `digest_matched`, never elevates tier/`checksum_verified`/consent — ADR-005, peer-review B-001), release create/upload for `publish` (new; `repo`-scope credential **resolved server-side** from daemon env/vault binding for the native tool, or read from the CLI process env for the CLI-local verb — never carried in tool inputs, payloads, errors, or events; peer-review B-005). Errors surface as diagnostics with suggested commands; anonymous rate-limit responses map to a named error advising a token.
- **git** (`internal/registry/gitsrc`, new) — `git` executable via `os/exec` with context timeout, shallow clone at ref into a temp dir, no credential handling in MVP (public repos; private via ambient git credentials). Absent `git` binary → deterministic `ErrGitUnavailable` diagnostic.

## Delete Targets

Per L-006, everything that disappears, in the same changes that replace it:

- `sdk/create-extension/` — entire npm workspace (templates move into the binary as `compozy extension init` embedded FS).
- `sdk/examples/secret-guard/`, `sdk/examples/telegram-reference/` — relocated to `internal/extension/testdata/`-backed E2E fixtures; `sdk/examples/` may contain public-SDK consumers only.
- `Manifest.Actions` (`[actions]`), `Manifest.Security` (`[security]`), `ActionsConfig`, `SecurityConfig` — replaced by `PermissionsConfig` (ADR-004).
- SDK `ExtensionDefinition.Actions/Security` (Go) and the TS equivalents; `validateProvidedMethodCoverage` hand map (`sdk/go/types.go:406-411`); `REQUIRED_PROVIDES_METHODS` hand map (`sdk/typescript/src/capabilities.ts:8-13`) — replaced by generated `RequiredMethods`.
- `[extensions.marketplace]` config section (`ExtensionsMarketplaceConfig`) — replaced by `[extensions.trust]` + `[extensions.sources]` (ADR-007); validation names the replacement key on legacy input.
- Shape-only capability validation as the acceptance path (`validateDottedIdentifiers` remains a lexer; membership becomes mandatory).
- Phantom capability strings everywhere they are taught: `prompt.provider`, `content.validate`, `message.read`, `message.write` in templates, `sdk/examples/prompt-enhancer`, `develop.mdx`.
- Hand-authored `min_compozy_version` in scaffolds/examples (build stamps it; the 17 stale `0.5.0` manifests are regenerated or corrected).
- `extensionHookSource = hookspkg.HookSourceConfig` aliasing (`internal/extension/manager.go:48`) — replaced by `HookSourceExtension`.
- npm `"private": true, "license": "UNLICENSED"` posture on `@compozy/extension-sdk`.
- CLI single-item jsonl gap (`extensionBundle`/`extensionRemoveBundle`/`extensionProvenanceBundle` get jsonl funcs — the "jsonl formatter is required" failure path for these verbs disappears).
- `contract.InstallExtensionRequest` slug/path shape — replaced by the source-union shape.
- `README.md:160` `compozy extension inspect` (nonexistent verb) — corrected to real verbs.

## Safety Invariants

1. Only `compozy extension dev` creates `extension_dev_links` rows; `install` can never mint a dev link (dev is a lifecycle overlay, not an `InstallSourceKind`), and dev links never carry marketplace trust fields.
2. Workspace binding is server-side only: `workspace_id` for dev/reload/logs is bound from the authenticated caller scope (operator: resolved workspace; agent: trusted session scope via the canonical dispatcher) and is never accepted from request bodies or tool inputs; agent-facing list/status/logs/SSE/event projections filter by that identity — no cross-workspace read or mutation path exists (L-033). Runtime dev state — subprocess, operation coordinator, last-good generation, log ring, status, events — is keyed by extension instance `(name, workspace_id)` with the global published installation as `(name, "")`; agent callers reach only instances their workspace resolves to, and the global instance's logs are operator-transport-only (R2 B-001).
3. Dev origin paths are canonicalized (`realpathDeepestExisting` + `EvalSymlinks` containment) and must resolve inside the bound workspace root at link time and at every load; a missing/escaping origin marks the extension `errored (missing_origin)` — the daemon never crashes boot and never runs a binary outside the recorded generation dir. Generation identity crosses transports only as a content-hash handle: the daemon validates the hash format, reconstructs the generation path itself under the canonicalized origin's output root (`dist/gen-<hash>`), and re-verifies the generation's manifest and content digest before activation — a caller-supplied path can never select execution content (R2 B-002).
4. Published installs verify the archive digest **before** any registry write; failure leaves zero partial state (existing `rollbackFailedInstall` path extended to all sources).
5. A colocated digest sidecar is integrity-only: a match records `digest_matched` and never elevates tier, `checksum_verified`, badges, or consent requirements; a mismatch aborts with `ExtensionArchiveDigestMismatchError` with no consent override. `checksum_verified` is exclusively catalog-pinned curated installs.
6. Closed-set validation: an unknown `capabilities.provides` or `permissions.requires` value fails manifest load/validate/install — a no-op load is impossible. Installed manifests declaring `bridge.adapter` are rejected deterministically until the bridge program lands (ADR-006); the public provide set is `tool.provider`, `memory.backend`, `model.source`, `loop.watch_source`.
7. Build/stage/swap/reload/watch for one extension instance serialize through a single per-instance operation coordinator (`InstanceKey`, R2 B-001): builds land in immutable generation directories, a generation is fully validated before the link's pointer swaps atomically, and concurrent operations coalesce — a torn or partially replaced bundle is unobservable by construction.
8. Failed activation restarts the **last-good generation**: the previous generation keeps running (or is restarted), status reports `errored (activation_failed; running <prior generation>)` with the failure in `last_error` and logs — never a half-started subprocess, never silent downtime.
9. `trusted_workspace` and `invocation_id` propagate daemon → SDK handler verbatim in both languages; SDKs never synthesize or default them.
10. Generated contracts — including the describe schema and required-methods maps — are byte-checked in CI for both languages (`make codegen-check`); a handwritten edit to generated files fails the gate.
11. Manifest generation is deterministic: identical source produces byte-identical `extension.toml` (sorted keys, stable ordering) — digest stability is a tested invariant.
12. Describe mode runs only in author-owned flows (`init|build|validate|dev|publish`); installing a published archive never executes extension code at install time. Native tools that execute extension or author code (`extensions_build`, `extensions_dev`, `extensions_reload`) are risk-classified and interaction-gated; `extensions_publish` is additionally `open_world`; `extensions_validate` is read-only.
13. Native tool inputs never contain credential values; the publish credential is resolved server-side from an approved binding (daemon env / vault) and never appears in tool transcripts, structured errors, events, SSE, or logs.
14. Subprocess stderr is redacted **at ingestion**: configured secret values (resolved `secret_env`, provider tokens) are masked before any byte enters the ring buffer or any broadcaster — redaction happens once, upstream of every transport (HTTP, UDS, SSE, native tools, web).
15. The log ring buffer is bounded (256 KiB per extension); buffers drop oldest lines and never block the subprocess.
16. A contributed command is never an execution authority: `extension exec` performs exactly one `POST /api/tools/:id/invoke` per invocation, so tool policy, approvals, risk gates, availability, and `trusted_workspace` apply unchanged — the CLI never calls an extension subprocess directly and never bypasses a gate the equivalent `tool invoke` would enforce.
17. Command paths are validated closed at build: `/`-joined, maximum depth 2, unique per extension, groups non-executable, flag names outside the reserved host set (`cmd`, `input`, `output`, `o`, `json`, `help`), and every mapped flag resolves to an existing top-level input-schema field — a malformed command tree fails the build, never the runtime. Declared groups reject leaf-as-parent, group/leaf collisions, empty segments, leading/trailing slashes, duplicates, multi-segment group paths, and leafless groups (R2 B-005); flag projection accepts only the closed schema subset — scalars, nullable scalars, enums of scalars, arrays of one scalar after canonical `$ref` resolution — and rejects composed/object schemas at build **and** at manifest load with an `--input` remediation (R2 B-006).

## Extensibility Integration Plan

- **Extension manifests**: schema v2 (`permissions` replaces `actions`/`security`; generated for subprocess extensions). `extensions/dev-cycle` + all 8 `extensions/bridges/*` manifests regenerate/re-author in the same change (delete targets above).
- **Hooks**: new `HookSourceExtension` value + priority tier 300; `HookDecl`/introspection surfaces (`compozy hooks list`) expose the source truthfully; ordering suites extended.
- **Skills / capabilities**: skill surfaces untouched; glossary gains Permissions/Provides entries and the capability disambiguation note; official skill gains authoring reference (Web/Docs Impact).
- **Tools / resources**: `ExtensionToolRuntimeDescriptor` gains the optional `command` presentation block (ADR-008 — same category as the existing `friendly_verb`/`preview` fields; execution semantics unchanged); `ExtensionToolCallRequest` mirrors gain `TrustedWorkspace`/`InvocationID` in both SDKs; reason codes get a documented reference page; surfaces registry (`internal/extension/surfaces`) unchanged in shape.
- **Commands (new contributed surface)**: extensions contribute operator verbs by marking a tool with `command` metadata; groups are declarable presentation nodes (`ext.commandGroup`) traveling the full generated path — `DescribePayload.command_groups` → manifest `[[resources.command_groups]]` → read model (R2 B-005). No new capability, protocol method, registry, or authority — `compozy extension exec` resolves the descriptor and calls the existing `POST /api/tools/:id/invoke`. This closes the CLI-verb item that ADR-006 filed as Future (Linear AGH-105 updated; ADR-006 carries the supersession note). **No product command ships in this program**: the surface is proven by E2E fixture extensions, and the first product command — the dev-cycle `archive` port — belongs to the `cmd-archive` follow-up program (`.compozy/tasks/cmd-archive/_brief.md`, R2 B-007).
- **Bundles / registries**: bundle surfaces unaffected; `internal/registry` gains `gitsrc` + digest sidecar verification + release upload; marketplace curated feed unchanged (3 kinds).
- **Bridge SDKs**: no code change beyond the install-time gate; public positioning updated per ADR-006. `bridge.adapter` is excluded from the public completeness surface (templates, public harness matrix, docs' supported-provides list); installed manifests declaring it are rejected deterministically. Generated contracts still emit the bridge service methods as internal groundwork the follow-up bridge program consumes; in-tree bridge packages are unaffected.
- **MCP sidecars**: manifest `resources.mcp_servers` unchanged; no impact (checked: `internal/mcp` host-API facade wiring untouched).
- **Protocol docs**: describe-mode contract, dev-lane lifecycle, permissions model, and source-union install documented in the extensions docs set.
- **Test harness**: public `sdk/go/extensiontest` + existing TS `@compozy/extension-sdk/testing` covering the **four public provides** (`tool.provider`, `memory.backend`, `model.source`, `loop.watch_source`); bridge-adapter conformance remains `internal/extensiontest` for in-tree bridges until the bridge program lands (ADR-006 / peer-review B-008).

## Agent Manageability Plan

- **CLI verbs** (all with `-o json|jsonl|toon`; jsonl parity fixed for single-item verbs): `init`, `build`, `validate` (structured `issues[]` + derived consent areas), `dev`, `reload`, `logs` (`-f` follows), `install` (source union), `update` (`--all` reports per-item partial progress from the daemon's `UpdateBatch`), `remove`, `enable`, `disable`, `list` (gains `Update` column), `status` (gains failure counters + one-line derived summary), `provenance`, `search` (source union + `--cursor`), `publish`, `commands` (contributed-command tree), `exec` (`<ext> --cmd <verb-path>` + projected flags or `--input`).
- **HTTP/UDS parity**: every daemon-backed verb has both transports (table in API Endpoints); daemon-free verbs (`init|build|validate|publish`) are CLI-local by design and callable by agents via the native tools below.
- **Native tools** (9 new, joining the existing 7 — the full authoring loop with no shell-out gaps, peer-review B-004): `compozy__extensions_init`, `compozy__extensions_build`, `compozy__extensions_validate`, `compozy__extensions_dev`, `compozy__extensions_reload`, `compozy__extensions_logs`, `compozy__extensions_search`, `compozy__extensions_provenance`, `compozy__extensions_publish`. Init/build reuse the same daemon-free services as the CLI verbs (scaffold templates, `BuildBundle`) exposed with identical structured outputs and deterministic errors. Risk classes (peer-review B-005): `validate` read-only; `init` low; `build`/`dev`/`reload` execute author/extension code → `requires_interaction` (approval-gated); `publish` additionally `open_world`. Toolset description updated to match actual membership.
- **Deterministic errors**: `ExtensionStatusCode` mapping extended (dev origin missing 409, reload without an active dev link 409, invalid/stale generation hash 400, validation 400 with issues payload, git unavailable 503-class diagnostic); every authored diagnostic (11 sites + new ones) reaches the human terminal via `renderHumanExecutionError` and structured output unchanged.
- **Status/config discovery**: `compozy extension status -o json` carries failure counters + backoff; `compozy config set` registry covers every `[extensions.*]` key; `compozy doctor` gains the extension probe (manifest compat, missing env, crash-loop, stale dev origin).
- **Docs for the agent path**: `skills/compozy/` authoring + management reference (see Web/Docs Impact).

## Config Lifecycle

Per ADR-007 — all in the same change as the code:

| Key | Default | Notes |
| --- | --- | --- |
| `extensions.trust.allow_unverified` | `true` | Operator policy; per-install consent still mandatory. Replaces `extensions.marketplace.allow_unverified` (deleted). |
| `extensions.sources.github.enabled` | `true` | Replaces `extensions.marketplace.registry` (deleted). |
| `extensions.sources.github.base_url` | `https://api.github.com` | Replaces `extensions.marketplace.base_url` (deleted). |
| `extensions.sources.git.enabled` | `true` | New. |
| `extensions.dev.watch_interval` | `2s` | CLI-side poll cadence for `dev --watch`. |
| `extensions.resources.*` | existing | Unchanged shape — now documented in config-toml reference. |
| `marketplace.catalog.*` | existing | Unchanged (cross-kind curated feed). |

Structs, defaults, merge/overlay (`merge_extensions_marketplace.go` replaced), validation (legacy keys rejected with replacement named in the error), `internal/cli/config_extensions.go` key registry, `config.toml` examples, generated CLI docs, and site config reference all move together; tests per `_tests.md`.

## Web/Docs Impact

- **`web/src/systems/marketplace`**: render the built-but-unrendered update affordance (`use-extension-detail-state.ts` → `marketplace-detail-extension-manage.tsx`); update counts across scopes (not only `installed` — `use-marketplace-kind-page.ts:200,256`); local-path + github/git install forms (contract already carries `path`; request shape changes with the union); source + verification badges on cards; empty-state `cliHint` updated to real verbs.
- **`web/src/systems/extensions`**: dev-source badge + `overrides published` label; failure counters/backoff + origin path in detail; logs panel consuming `GET /api/extensions/{name}/logs` (SSE follow); provenance dialog gains source kinds. Truthful-UI: every new control maps to a shipped route above; nothing rendered the daemon doesn't model.
- **Generated types**: `make codegen` refreshes `openapi/compozy.json` + `web/src/generated/compozy-openapi.d.ts`; E2E mocks/matchers co-ship (L-007).
- **`packages/site`**: `develop.mdx` full rewrite around the code-first journey (init → dev → reload → publish); new quickstart under getting-started/guides; permissions + consent-areas reference; publish guide; manifest v2 reference (single source of truth page); **contributed-commands guide** (declaring a command and groups, `/`-nested paths and depth limit, the closed flag-projection subset, `--input` escape, `exec` vs `tool invoke`, a worked example on a sample extension — the dev-cycle `archive` walkthrough lands with the `cmd-archive` program); `config-toml.mdx` new `[extensions.*]` table incl. `resources.*`; CLI reference regen (9 new verbs, changed flags); hook event catalog corrected (claims match content; extension source documented); tool-ID grammar (`ext__<extension>__<tool>`) and the command-path grammar specified; bridge-wall paragraphs adopt ADR-006 positioning; migration page notes the hard cuts.
- **`skills/compozy/`**: new authoring reference (manifest v2, permissions, dev loop, publish, native tools) wired into the router table — a Compozy agent can build a Compozy extension.
- **Root `README.md`**: `extension inspect` corrected; quickstart pointer.
- **QA tracker**: user-visible behavior changes throughout → new/reset content-addressed scenarios in `docs/qa/scenarios/` per phase (flag, don't retest); authoring-path scenarios added (scaffold→build→validate→dev→reload→publish→install→invoke→update→remove).

## Impact Analysis

| Component | Impact | Description / Risk | Action |
| --- | --- | --- | --- |
| `internal/extension` | modified (heavy) | manifest v2, build/dev/publish/logs, trust simplification — highest-risk surface | phased tasks + invariants 1–8, 11–15 |
| `internal/extension/contract` | modified | describe payload, command/group specs, consent table sources (single authority) | co-evolve with codegen |
| `cmd/compozy-codegen`, `internal/codegen/sdkgo` | new target | generated Go contracts; drift gate | mirror `sdkts` structure; check mode |
| `sdk/go` | restructured | nested module + generated layer; release tag `sdk/go/vX.Y.Z` | boundaries test + release wiring |
| `sdk/typescript` | modified | describe mode, generated map, dropped-fields fix, npm publish | contract tests + publish in `release.yml` |
| `sdk/create-extension` | **deleted** | scaffolder moves into binary | embedded templates in `internal/cli` |
| `internal/cli` | modified | 9 new verbs (incl. `commands`/`exec`), flag projection, error rendering, output polish | golden-output tests |
| `internal/api/*` | modified | 5 new routes, request hard cut, payload additions | contract co-ship (L-007) |
| `internal/daemon` | modified | wiring, 9 new native tools, search union, doctor probe | composition-root only |
| `internal/hooks` | modified | `HookSourceExtension` + tier 300 | ordering suite extension |
| `internal/registry` | modified/new | `gitsrc`, digest sidecar, release upload | mocked-API integration tests |
| `internal/config` | modified | ADR-007 sections; legacy keys rejected | lifecycle tests |
| `internal/store/globaldb` | migration | extensions table columns (hard cut) | `eng-schema-migration` suites |
| `web/` | modified | marketplace/extensions surfaces above | turbo lane + Playwright |
| `packages/site`, `skills/compozy`, `README` | modified | docs program | docs build + verbatim-follow QA |
| `extensions/{dev-cycle,bridges/*}` | modified | manifests regenerate to v2 (no product command here — the ADR-008 surface is fixture-proven; dev-cycle `archive` belongs to `cmd-archive`) | in-repo fixture updates |
| `.github/workflows/release.yml` | modified | npm publish + sdk/go tag | dry-run verification |
| `magefiles/boundaries.go` | modified | new packages registered | same-commit rule |

## Testing Approach

- **Unit** (`go test -race`, scoped per package; Vitest for TS SDK): manifest v2 validation/closed sets, consent derivation, build determinism, dev link/reload state machine, trust gate matrix, source parsing union, digest sidecar verification, CLI output/error rendering golden tests, generated-contracts compile + map correctness in both languages.
- **Integration** (`+integration`): registry migrations (fresh/reopen/ahead/integrity/equivalence), install pipelines per source against mocked GitHub/git fixtures, describe-mode subprocess round trip, log ring buffer under load.
- **E2E runtime** (Go harness): the authoring loop end to end on a stamped version binary — init → build → validate → dev → invoke tool → reload → publish (mock GitHub) → install-from-release → update → remove; agent-driven variant through native tools with structured output only.
- **E2E web** (Playwright): update affordance, dev badge, install-from-path/github forms, logs panel.
- **Cross-build**: `GOOS=windows GOARCH=amd64 go build` gate for subprocess/dev-link paths.
- Fakes only at I/O boundaries (GitHub API, git binary, subprocess transport); the E2E lane uses real daemon + real SDK subprocesses. Every concrete case with IDs lives in `_tests.md`, whose **Suite Placement** section maps every case group to its owning canonical suite (existing file, or a justified new suite) per `eng-consolidate-test-suites` (peer-review B-009) — placement is decided in the contract, not at implementation time.

## Development Sequencing

### Build Order

1. **Phase A — Contracts & publishing groundwork**: `sdk-contracts-go` target + generated layer; single-sourced `RequiredMethods` replacing both hand maps; `TrustedWorkspace`/`invocation_id`/`model.source` parity in both SDKs; `sdk/go` nested `go.mod`; npm/module publish wiring in `release.yml`. *Gate*: `make codegen-check`, scoped `go test`, `bun-test`, boundaries.
2. **Phase B — Manifest v2 + permissions**: `PermissionsConfig`, closed-set validation, consent derivation, `HookSourceExtension`, registry schema migration, `extensions/*` in-repo manifests regenerated. *Gate*: migration suites + scoped tests.
3. **Phase C — Code-first toolchain**: describe mode in both SDKs; `BuildBundle`/`ValidateBundle`; `init` (embedded templates: tool-provider TS/Go, hook TS, memory-backend TS, watch-source Go), `build`, `validate` verbs; `min_compozy_version` stamping + CI gate keying in-repo manifests to the daemon version. *Gate*: build-determinism test + e2e build→install fixture.
4. **Phase D — Dev lane**: `LinkDev`/`UnlinkDev`/`ReloadExtension`; `dev|reload|logs` verbs + `--watch`; routes + UDS + native tools; log ring buffer. *Gate*: e2e dev loop (edit → reload → observe).
5. **Phase E — Distribution**: source-union install contract; `gitsrc`; digest sidecars; `publish` verb + release upload; search union route/verb; provenance `git_url`/github truthful; update flows for github/git sources with batch partial progress exposed. *Gate*: integration with mocked GitHub + e2e publish→install.
6. **Phase F — UX & surfacing**: human-mode diagnostic rendering; success output + next-step hints; `update_available` in list/search/web; status summary + failure counters; doctor probe; jsonl parity; path-typo fix (stat errors surface); **contributed commands** (`ExtensionCommandSpec`/`ExtensionCommandGroupSpec` in `internal/extension/contract` + generated SDK support, manifest `command` block + `[[resources.command_groups]]`, build-time validation incl. the closed flag subset, `GET /api/extensions/commands` + UDS, `extension commands`/`extension exec` with two-phase flag projection, E2E fixture extension proving the surface — no product command); web changes. *Gate*: CLI golden tests + turbo web lane + Playwright.
7. **Phase G — Docs, skill, examples, QA tail**: site rewrite + quickstart + references; `skills/compozy` authoring; `sdk/examples` cleanup + new installable examples; README fix; glossary entries; QA scenario flags + authoring scenarios; then the standard `qa-report`/`qa-execution` pair closes `_tasks.md`. *Gate*: full `make verify` + docs build.

### Technical Dependencies

A → B (generated constants feed closed-set lists) → C (build emits manifest v2) → D (dev consumes build) → E (publish consumes build; install consumes union). F depends on B/D/E payloads; G depends on everything. npm/Go publishing requires release-workflow access; GitHub integration tests run against mocks (no live API in CI).

## Monitoring and Observability

- Canonical events (append-only store) with a **per-event required-key matrix** (R3 N-003 — keys are required only where meaningful; no empty or invented values):

  | Event | Required correlation keys |
  | --- | --- |
  | `extension.install.completed\|failed` | `extension_name`, `source_kind`, `digest_matched` |
  | `extension.update.completed\|failed` | `extension_name`, `source_kind` |
  | `extension.dev.linked\|unlinked` | `extension_name`, `workspace_id`, `bundle_generation` |
  | `extension.reload.completed\|failed` | `extension_name`, `workspace_id`, `bundle_generation` |
  | `extension.publish.completed\|failed` | `extension_name` |
  | `extension.crash_loop.backoff` | `extension_name`, plus `workspace_id` when the crashing instance is a dev instance (existing failure path now emitting counters) |
- Coverage-matrix test extended so **every** success and failure path above emits its event with its per-event required keys from the matrix, asserted through a test notifier/event sink at the emitting call site (peer-review N-003/B-009, R3 N-003); the same assertions verify no secret value appears in any event payload (invariants 13–14).
- Log fields: `slog` with `extension`, `source`, `phase`, `reason_code`; reason codes get a public reference page (docs) so surfaced codes are documented.
- Metrics: install count by source/verified, reload latency, crash-loop disables, search fan-out latency.

## Technical Considerations

### Key Decisions

Recorded as ADRs (listed below). Additional in-spec decisions: **update discovery** is projection-only (no stored state, no background poller — TTL'd catalog + on-list registry lookups; peer-review N-004 contract: per-source lookup bounded at 2s, non-curated results cached in-memory for 15m, a rate-limited/unavailable source degrades to cached-or-omitted results with a `sources_degraded` marker on the payload — listing never blocks on a dead source); **watch** is CLI-side (daemon stays passive toward author dirs); **scaffolder** is the binary (`init` with embedded templates — zero-dependency entry; an npm `create` wrapper is post-MVP); **protocol version** stays strict `"1"` (hard-cut in place per L-025; revisit at GA — ADR-003); **status rendering** derives one summary line from the four axes instead of collapsing the axes in storage.

### Known Risks

- **Nested `sdk/go` module mechanics** (tagging, internal `replace`, proxy behavior) — mitigated by release dry-run task in Phase A and CI consuming the SDK as an external module in one test job.
- **Anonymous GitHub rate limits** degrade search/install UX — deterministic diagnostic advises `GITHUB_TOKEN`; curated tier unaffected.
- **Describe-mode portability** (author toolchains vary) — detection covers package.json/go.mod; `--cmd` is the escape hatch; timeout bounds hangs.
- **Windows parity** for dev links + subprocess reload — cross-build gate + `internal/procutil` centralization; no symlinks used (link = registry row, not FS symlink).
- **Migration blast radius** on the extensions table — alpha wipe/reinstall accepted and documented; fresh/reopen suites cover both.

## Assumptions / Defaults

- `[extensions.sources.github].enabled = true` and `[extensions.trust].allow_unverified = true` by default; consent remains per-install; locked-down operators flip two keys.
- Dev watch default `2s` poll (filesnap pattern); no fsnotify dependency introduced.
- Publish requires `GITHUB_TOKEN` (repo scope); no `gh` CLI dependency.
- Alpha data policy: extension registry hard cut → users re-link dev extensions and reinstall published ones; no data migration shims.
- Curated catalog stays the cross-kind feed; skills stay on ClawHub this program (ADR-005).
- Bun is the default TS toolchain in scaffolds; Go builds use the module's own toolchain.
- The brief's scorecard targets bind Phase G acceptance: concepts ≤10, first-success actions ≤4, own-install ≤1 command + 1 consent, SDK re-grades ≥ A−/B/B.

## Architecture Decision Records

- [ADR-001: Code-first authoring — manifest as build artifact](adrs/adr-001.md) — SDK is the single source; `build` generates the manifest; drift dies by construction.
- [ADR-002: First-class dev lane](adrs/adr-002.md) — `dev`/`reload`/`--watch` with link semantics; trust ceremony removed from author-owned code.
- [ADR-003: Uniform SDK architecture](adrs/adr-003.md) — thin transport + generated contracts + minimal ergonomics; both SDKs published.
- [ADR-004: Single permissions list](adrs/adr-004.md) — `permissions.requires` closed-set; consent areas derived; `security.capabilities` deleted.
- [ADR-005: GitHub/git-first distribution](adrs/adr-005.md) — source union, digest sidecars, `publish` verb; skills/ClawHub divergence recorded.
- [ADR-006: Closed-surface positioning](adrs/adr-006.md) — bridges = declared follow-up program; four surfaces Future (Linear AGH-105).
- [ADR-007: Extension config consolidation](adrs/adr-007.md) — `[extensions.trust]`/`[extensions.sources]`; `[extensions.marketplace]` deleted.
- [ADR-008: Extension-contributed commands](adrs/adr-008.md) — command metadata on a tool + `compozy extension exec <ext> --cmd <path>`; `/`-nested depth 2; zero new authority; fixture-proven here — the dev-cycle `archive` port ships with the `cmd-archive` program.

## References

- Audit corpus: `.compozy/tasks/ext-improvs/_brief.md` (R1–R11 + evidence file:line index).
- Competitor references (implementers read these): `.resources/zed/crates/extension_host/src/extension_host.rs:1032-1133`, `.resources/zed/crates/extension_cli/src/main.rs:27-90`, `.resources/claude-code/utils/plugins/{schemas.ts:906-1000,validatePlugin.ts,parseMarketplaceInput.ts}`, `.resources/claude-code/commands/reload-plugins/reload-plugins.ts`, `.resources/openclaw/docs/tools/{plugin.md,clawhub.md}`, `.resources/openclaw/docs/plugins/manifest.md:29-31`, `.resources/pi/packages/coding-agent/{docs/extensions.md,docs/packages.md,src/core/extensions/types.ts:1185-1412}`, `.resources/eve/docs/extensions.md`, `.resources/eve/packages/eve/src/setup/scaffold/create/extension.ts:48-180`, `.resources/eve/scripts/extension-capability-contracts.mjs:38-74`, `.resources/flue/blueprints/README.md:52-78`, `.resources/flue/packages/cli/src/commands/blueprints.ts:169-253`.
- Current-shape inventory: `internal/extension/{manifest.go,manager.go,registry.go,registry_types.go,marketplace_trust.go,provenance.go,tool_reconciliation.go,capability.go,surfaces/registry.go}`, `internal/cli/{extension.go,extension_state.go,extension_output.go,format.go,root.go}`, `cmd/compozy-codegen/main.go`, `internal/codegen/sdkts/`, `internal/hooks/{types.go,ordering.go}`, `sdk/go/`, `sdk/typescript/`, `internal/config/{extensions_marketplace.go,marketplace.go}`, `internal/daemon/{extensions.go,native_extension_tools.go,extension_manager_wiring.go}`, `internal/api/core/extensions.go`, `internal/registry/`, `internal/tools/{provider_descriptor.go,reason.go}`, `internal/filesnap/`.
- Memory: `docs/_memory/standing_directives.md` (SD-002/006/008/011), `docs/_memory/lessons/{L-001,L-003,L-006,L-007,L-008,L-012}.md`, `docs/_memory/glossary.md`.
