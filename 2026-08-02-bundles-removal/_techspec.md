# TechSpec: Bundles Removal — Extension as the Single Kit Unit (bundles-removal)

Companion test contract: `_tests.md`. Requirements source: `_brief.md` (no PRD exists for this program; the brief is the requirements document). ADRs: `adrs/adr-001..008.md`. Baseline: the `ext-improvs` branch (Extension DX Overhaul — `.compozy/tasks/ext-improvs/_techspec.md` + ADRs); this spec's file:line evidence was verified against that worktree, which corrects several paths in the brief (site docs live under `packages/site/content/runtime/**`, nav is `packages/site/lib/runtime-navigation.ts` + `meta.json` files, and there is no production `registry_bundles.go` — that logic lives in `registry.go`/`registry_lifecycle.go`).

Requirement keys used throughout and in `_tests.md`: **R-HC-1..7** (brief "Hard cut" 1–7), **R-P0-1..5** (Extension robustness P0), **R-P1-1..5** (P1), **R-AM-1..4** (Agent-manageability & co-ship).

## Executive Summary

This program hard-deletes the public Bundle product and makes Extension the only kit unit. Four architectural moves carry the design: (1) **kit completeness on enable** — static extension resources grow to cover everything bundles projected: dir-per-agent agents with parsed `SOUL.md`/`HEARTBEAT.md` sidecars (ADR-007), package automation jobs/triggers with enable-as-consent (ADR-002), package window layouts, all published with `Owner{Kind:"extension"}` at global scope (ADR-004) and declared code-first through describe/build (ADR-008); (2) **MCP-shaped secrets binding** — an instance-keyed `extension_env_bindings` side table (`(extension, workspace_id, env)`; `''` = the global published instance), the first producer of the `vault:extensions/...` namespace (`global/` and `ws/<workspace>/` ref prefixes, the exact MCP dual convention), and `compozy extension secrets set|bind|list|unset` with transactional writes, presence projection, `requires_env`-intersected injection, and bindings that survive update under the same managed identity (ADR-003); (3) **lifecycle-native network consent** — the manifest's existing `network_participation` digest gates `enable`, `update` refuses **before** applying when the digest changes unless the caller confirms the exact new digest, dev-link instances persist their own confirmation state, and a daemon-owned per-extension lifecycle coordinator serializes every lifecycle mutation end to end (ADR-005); (4) **dedicated read-only operability** — `extension inventory` (shipped vs live) and `extension preview` (enable dry-run: publish set, agent conflicts, unbound env, automation that would start, network digest) as CLI verbs + unprivileged GET routes + read-only native tools (ADR-006). With the replacements live, the cut removes `internal/bundles/**` (33 files), the bundle schema/loaders in `internal/extension`, 8 HTTP/UDS routes and OpenAPI operations, the `compozy bundle` CLI, 5 native tools + 1 toolset + 2 capability strings, marketplace kind `bundle`, resource kinds `bundle`/`bundle.activation` (plus the `internal/resources` mixed-kind projector escape hatch they alone consume), the skills `installed_from_bundle` field, 11 dedicated web files + ~40 shared strips, 11 site doc files + ~25 section strips, and the QA/skill/ecosystem teaching of Bundle as a product (ADR-001).

Primary trade-offs: profile selection, workspace-scoped packaged resources, and bundle bridge presets are dropped without replacement (brief non-goals; bridge instances remain the separate `compozy bridge setup` operator step); `extension update` gains a consent refusal path; and two Goose migrations land (additive bindings/confirm schema, then one-shot orphan-row cleanup). No compat shims, no aliases, no dual fields, no placeholder stubs anywhere in this program — every replaced surface's old form is deleted in the same change that ships its replacement.

**MVP boundary:** phases A–F below are all in scope for this program's `_tasks.md`, closed by the standard `qa-report` + `qa-execution` pair. Post-MVP, explicitly out of scope: workspace-scoped enable, any pack/compose or multi-profile product, per-profile selective resource scoping, Live channel enrollment/memory seeding/task-graph starters, extension-projected declared channels, webhook-event package triggers, bridge SDK expansion, and a native tool for secret writes (parity with MCP's read-only tool posture).

## System Architecture

### Component Overview

- **`internal/extension` (modified, heavy)** — deletes the bundle schema and loaders (13 `bundle*.go` files) and `resources.bundles`; gains the dir-per-agent loader with sidecar parsing (rehomed from `bundle_agent_loader.go`), automation + layout static loaders (rehomed validation from `bundle_job.go`/`bundle_layout_loader.go`), env-binding resolution in `resolveEnvMap`, network-confirm state on the registry, inventory/preview computation over the manager snapshot, and describe/build support for declared resource dirs (ADR-008). `ErrExtensionHasActiveBundles` and its direct-SQL activation scan are deleted.
- **`internal/extension/contract` (modified)** — `DescribePayload` gains `resources`; generated into both SDKs (byte-checked).
- **`sdk/go` + `sdk/typescript` (modified)** — definition-level `resources` declaration; regenerated contracts.
- **`internal/vault` (modified)** — `extension_refs.go`: `ExtensionSecretOwnerPrefix`/`ExtensionSecretRef` (mirror of `mcp_refs.go`); ownership kind `extension_env`.
- **`internal/store/globaldb` (modified)** — migration `00029` (add `extensions` confirm columns + `extension_env_bindings` table), migration `00030` (one-shot cleanup of orphaned bundle rows), sqlc queries for bindings; deletes `global_db_bundles.go` + `ListBundleActivationResourceSpecs`.
- **`internal/daemon` (modified)** — deletes `bootBundles`, the bundle publisher/projection/store files, and `native_bundle_resource_tools.go`; extends the agent/skill publisher with soul/heartbeat desired resources + extension owner; adds automation and layout declaration providers (cloning the loop provider pattern); adds the secrets binding service (transactional, MCP-patterned, instance-keyed), the **per-extension lifecycle mutation coordinator** (the single authority serializing preflight → registry/filesystem mutation → confirmation write → reload → reconcile → response, with defined commit point and reverse rollback — B-002), confirm gates in enable/update, inventory/preview services, and 2 read-only native tools; overlay GC when package automation definitions disappear.
- **`internal/api/{contract,core,spec,httpapi,udsapi}` (modified)** — deletes the bundle contract/handlers/spec/routes and the marketplace `bundle` kind; adds inventory/preview/secrets payloads, routes, and error mappings; enable/update requests gain `confirm_network_digest` (the exact digest being consented to, never a bare boolean); enable returns a dedicated `ExtensionEnableResult`; `ExtensionPayload` −`bundles` +`bound_env_keys` + network-confirm fields; skill payloads −`installed_from_bundle`.
- **`internal/cli` (modified)** — deletes `bundle*.go` + `newBundleCommand` + bundle client methods; adds `inventory`, `preview`, `secrets set|bind|list|unset` verbs, `enable`/`update` `--confirm-network-requirement <digest>` flags, enable output rendering `ExtensionEnableResult` (started automation enumerated).
- **`internal/tools` + `internal/toolmeta` (modified)** — deletes `compozy__bundles_*`, toolset `compozy__bundles`, capabilities `bundles.read`/`bundles.write`; adds `compozy__extensions_inventory`/`compozy__extensions_preview` (read-only; wildcard toolset `compozy__extensions_*` absorbs them); `compozy__extensions_enable` input gains `confirm_network_digest`.
- **`internal/resources` (modified)** — deletes `BundleActivationProjector` and the mixed-kind registration seam (only consumer gone); test fixtures using `"bundle"` kinds renamed to neutral kinds.
- **`internal/skills` / `internal/automation` / `internal/bridges` (light)** — delete `InstalledFromBundle` (field, spec round-trip, payload, CLI output); reword `JobSourcePackage`/`BridgeInstanceSourcePackage` comments to "extension package"; keep automation overlay + read-only semantics unchanged.
- **`web/`, `packages/site`, `skills/compozy`, QA tree** — per Web/Docs Impact below.

Data flow (kit path after this program): author SDK definition (+ resource dirs) → `compozy extension build` → generation with declared `[resources]` → install/dev-link → `extension secrets set` (if `requires_env`) → `enable [--confirm-network-requirement]` → register loads agents+sidecars/skills/loops/automation/layouts → publishers write owner-attributed global records → catalogs/scheduler/window-manager consume → `disable` deletes the owned set (+ overlay GC).

## Architectural Boundaries

- **No new packages.** All new code lands in existing packages; `internal/bundles` is deleted, removing its 14 production importers (all in `internal/daemon` + `internal/api/core`).
- `internal/extension` keeps its import set — it already imports `internal/automation` (bundle job validation) and `internal/windowmanager` (layout codec); the new static loaders keep those imports for load-time validation. It must still NOT import `internal/daemon`, `internal/api/*`, `internal/cli`, or `internal/marketplace`.
- Composition stays in `internal/daemon` (SD-008): publishers, secrets service, confirm gates, inventory/preview wiring, native tools. `bootBundles` is deleted from `internal/daemon/boot_automation_bundles.go`; the surviving `bootAutomation`/`bootExtensions` content is re-filed by responsibility (`boot_automation.go`, existing `boot_extension*` files) — no file keeps a "bundles" name.
- `internal/vault` gains only the ref-namer; it does not learn about extensions beyond the namespace string (same relationship `mcp_refs.go` has to MCP).
- `magefiles/boundaries.go`: the `internal/marketplace ↛ internal/bundles` rule is removed **in the same commit** that deletes the package.
- Authoritative primitives unchanged (L-005): the scheduler still never claims; automation definitions stay read-only outside overlays; `resources` kernel remains the only record writer; enable/disable remain the only kit switches.

## Implementation Design

### Core Interfaces

Manifest v2 resource fields (bundles deleted, automation/layouts added; TOML + JSON forms):

```go
// internal/extension/manifest.go — ResourcesConfig after this program
type ResourcesConfig struct {
    Skills        []string                   `toml:"skills,omitempty"`
    Loops         []string                   `toml:"loops,omitempty"`
    Agents        []string                   `toml:"agents,omitempty"`     // dirs of agent dirs (ADR-007)
    Automation    []string                   `toml:"automation,omitempty"` // NEW: dirs/files of automation TOML (ADR-002)
    Layouts       []string                   `toml:"layouts,omitempty"`    // NEW: dirs/files of window-layout JSON
    Hooks         []HookConfig               `toml:"hooks,omitempty"`
    Tools         map[string]ToolConfig      `toml:"tools,omitempty"`
    CommandGroups []manifestCommandGroupSpec `toml:"command_groups,omitempty"`
    MCPServers    map[string]MCPServerConfig `toml:"mcp_servers,omitempty"`
    Publish       ResourceGrantRequest       `toml:"publish,omitempty"`
    // Bundles: DELETED (no alias, no fallback)
}
```

Static agent loader (rehomes `bundle_agent_loader.go` semantics; replaces the flat markdown walk):

```go
// internal/extension/agent_resources.go
type AgentSidecar struct{ SourcePath, Body string } // parsed+validated at load (soul.Parse / heartbeat.Parse)
type StaticAgent struct {
    Agent     compozyconfig.AgentDef // <entry>/<agent>/AGENT.md (mcp.json + capabilities.toml merged by LoadAgentDefFile)
    Soul      *AgentSidecar          // optional SOUL.md
    Heartbeat *AgentSidecar          // optional HEARTBEAT.md
}
// Rejects loose .md files directly under an entry; rejects a missing AGENT.md; malformed sidecars fail with file+reason.
func LoadAgentResources(rootDir string, paths []string) ([]StaticAgent, error)
```

Package automation loader (schema mirrors `BundleJob`/`BundleTrigger` minus profile nesting; validation via throwaway `automation.Job`/`automation.Trigger` with `Source: package`, exactly as `bundle_job.go` does today):

```go
// internal/extension/automation_resources.go — TOML files with [[jobs]] / [[triggers]]
type ExtensionJob struct {
    Name      string                    `toml:"name"`
    Agent     string                    `toml:"agent"`   // must name an agent SHIPPED BY THIS EXTENSION (self-contained kit; external/visible
                                                         // agents are not valid targets — operator-authored dynamic jobs cover that case) (R1 N-003)
    Prompt    string                    `toml:"prompt"`
    Schedule  automation.ScheduleSpec   `toml:"schedule"`
    Task      *automation.JobTaskConfig `toml:"task,omitempty"`
    Enabled   *bool                     `toml:"enabled,omitempty"` // default true; enable is the consent act (ADR-002)
    Retry     automation.RetryConfig    `toml:"retry,omitempty"`
    FireLimit automation.FireLimitConfig `toml:"fire_limit,omitempty"`
}
type ExtensionTrigger struct {
    Name, Agent, Prompt, Event string            // Event == "webhook" rejected (carried over from bundles)
    Filter                     map[string]string
    Enabled                    *bool
    Retry                      automation.RetryConfig
    FireLimit                  automation.FireLimitConfig
    EndpointSlug               string
}
func LoadAutomationResources(rootDir string, paths []string) ([]ExtensionJob, []ExtensionTrigger, error)
```

Layout loader (rehomes `bundle_layout_loader.go`): each declared path yields `windowmanager.LayoutResource` values decoded+validated by `windowmanager.NewLayoutResourceCodec().DecodeAndValidate` at global scope; `.json` regular files only, root-jailed.

Env bindings (ADR-003):

```go
// internal/vault/extension_refs.go — the exact MCP dual convention (global vs workspace instance)
func ExtensionSecretOwnerPrefix(extension, workspaceID string) string
// workspaceID "" → "vault:extensions/global/<ext>/"; else "vault:extensions/ws/<workspace>/<ext>/"
func ExtensionSecretRef(extension, workspaceID, envName string) string // prefix + "env/<NAME>" (unsafe segments hex-encoded)

// internal/extension/env_bindings.go — bindings are INSTANCE-keyed (R1 B-001): the published global
// instance is (name, "") and a dev-linked instance is (name, workspace_id), matching InstanceKey.
type EnvBinding struct {
    ExtensionName string
    WorkspaceID   string    // "" = global published instance; dev instances bind their workspace server-side
    EnvName       string    // must be a declared requires_env name; undeclared → deterministic error
    SecretRef     string    // namespace "extensions" enforced
    Kind          string    // "extension_env" ownership tag for GC
    CreatedAt, UpdatedAt time.Time
}
type EnvBindingStore interface {
    ListEnvBindings(ctx context.Context, extension, workspaceID string) ([]EnvBinding, error)
    PutEnvBinding(ctx context.Context, b EnvBinding) error
    DeleteEnvBinding(ctx context.Context, extension, workspaceID, envName string) error
}

// internal/daemon/extension_secrets.go — transactional service (MCP pattern: snapshot → put → reverse rollback → GC)
type SecretInput struct { // exactly one of Value/VaultRef per env name (R1 N-006); overlap is a 400
    Value    *string // plaintext (write-only; stored at the conventional instance ref)
    VaultRef *string // existing ref (namespace + metadata.Present checked; dangling → 400)
}
type SetExtensionSecretsRequest struct {
    Secrets map[string]SecretInput // env name → input; mutation order = sorted normalized env names; rollback reverse
}
func (s *extensionsService) SetSecrets(ctx context.Context, key extension.InstanceKey, req SetExtensionSecretsRequest) (*contract.ExtensionSecretsPayload, error)
func (s *extensionsService) DeleteSecret(ctx context.Context, key extension.InstanceKey, envName string) error
```

Spawn resolution order in `resolveEnvMap` (deterministic): safe allowlist → manifest `env` → manifest `secret_env` → **operator bindings of the launching instance** — the instance resolves exactly its own `(name, workspace_id)` rows, with no cross-scope fallback (a dev instance never inherits global bindings, and vice versa). Injection intersects bindings with the **current** normalized `requires_env` (R1 B-003): a binding whose declaration was dropped by an update is stale — reported by `secrets list` (`stale: true`) and never injected. Operator bindings override authored `secret_env` for the same NAME; every resolved secret still registers for redaction before use. `extension remove` (global instance) and `dev unlink` (workspace instance) delete the instance's binding rows and GC owned `extension_env` refs; bindings therefore survive `update` under the same managed identity but never survive remove→reinstall.

Network confirm (ADR-005):

```go
// internal/extension/registry_network.go — confirmation state is INSTANCE-scoped (R1 B-001):
// the published instance persists on the extensions row; a dev-linked instance persists on its
// extension_dev_links row (a dev-only extension has no extensions row and still has a home).
var ErrExtensionNetworkConfirmationRequired = errors.New("extension: network participation confirmation required") // wrapped with the CURRENT digest
type NetworkConfirmation struct{ Digest, ConfirmedBy string; ConfirmedAt time.Time }
func (r *Registry) NetworkConfirmation(key InstanceKey) (NetworkConfirmation, error)
// Confirmation requires the EXPECTED digest (R1 B-006): the caller consents to the digest it saw
// (from preview or the 409). The coordinator re-reads the candidate manifest under the lifecycle
// lock; expectedDigest != current → rejected with the current digest, never ratified.
// actor is the real caller identity: "operator" on operator transports, "agent:<session-scoped id>"
// via the native tool's canonical dispatcher — an agent confirm is never recorded as operator.
func (r *Registry) ConfirmNetworkRequirement(key InstanceKey, expectedDigest, actor string, at time.Time) error

// internal/daemon/extension_lifecycle.go — the lifecycle mutation coordinator (R1 B-002): ONE
// per-extension-name authority serializes enable/disable/install/update/remove/dev-link/reload
// end to end. No registry, filesystem, confirmation, or runtime mutation happens outside it.
//   withLifecycle(name): preflight (re-read manifest digest + registry version fence) →
//   staged registry/filesystem mutation → confirmation write → runtime reload → resource
//   reconcile → response construction. Commit point: the registry/filesystem swap. Rollback
//   (reverse order) restores the prior registry row, files, confirmation tuple, enabled bit,
//   and restarts the last-good generation; projections re-converge from restored state.
//   A refused or failed operation releases the coordinator with byte-identical persisted and
//   running state. The per-instance build/stage/swap coordinator from ext-improvs remains the
//   inner serialization; this wraps the full daemon-visible operation around it.
func (s *extensionsService) withLifecycle(ctx context.Context, name string, fn func(tx *lifecycleTxn) error) error

// Gates (inside the coordinator):
// Enable: manifest digest non-empty && stored instance confirmation invalid → 409 carrying the
//   current digest; request confirm_network_digest == current digest records {digest, actor, now}
//   atomically with the enable commit and proceeds; mismatch → 409 with the current digest.
// Update: staged new manifest digest != confirmed digest && request carries no matching
//   confirm_network_digest → refuse BEFORE any swap (zero changed state). `update --all` never
//   blanket-confirms: a digest-changing item is refused per-item with its digest in the batch
//   partial progress; confirmation happens per extension via a single update call.
// Dev reload: same gate against the dev link's confirmation state.
```

Inventory and preview (ADR-006):

```go
// internal/extension/inventory.go
type KitItem struct {
    Kind  resources.ResourceKind // agent | agent.soul | agent.heartbeat | skill | loop | tool | mcp_server | hook.binding | automation.job | automation.trigger | window_layout
    // Logical identity is (Kind, Name) — the shipped↔live join key (R1 N-002). ID is the live
    // record id when Live; for shipped-only items it is the source key the publisher would use
    // ("extension/<ext>/<kind>/<name>"), which is content-independent. A content change therefore
    // keeps one row (same Kind+Name) in inventory, and preview renders it as a changed item —
    // never a remove+add pair — while genuinely renamed resources do split into remove+add.
    ID    string
    Name  string
    Live  bool                   // a matching extension-owned record exists
}
type ExtensionInventory struct {
    Extension string
    Enabled   bool
    Items     []KitItem // shipped ∪ live, per kind, name-sorted
}
type EnablePreview struct {
    Extension                    string
    WouldPublish                 []KitItem
    AgentConflicts               []string // shipped agent names colliding with visible non-owned agents
    MissingEnv                   []string // requires_env minus (process env ∪ bound keys)
    AutomationStarting           []string // namespaced job/trigger names with effective enabled=true
    NetworkRequirementDigest     string
    NetworkConfirmationRequired  bool
}
```

Preview and enable compute the same desired state from the same snapshot (preview = compute; enable = compute + publish); an equivalence test pins that.

Enable action result (R1 B-005) — the automation enumeration is the consent record, so it is a dedicated **action result**, never status/list data (a later status read must not claim work "started"):

```go
// internal/api/contract/extensions.go — returned by POST /api/extensions/{name}/enable, rendered
// by the CLI verb and the native tool; identical across every transport.
type ExtensionEnableResult struct {
    Extension         ExtensionPayload `json:"extension"`
    AutomationStarted []string         `json:"automation_started"` // namespaced "<ext>/<name>", name-sorted;
    // exactly the effectively-enabled definitions (authored enabled × overlays) that became
    // runnable in THIS committed operation — not all authored definitions, not later state.
}
// internal/daemon — shared core service signature changes accordingly:
func (s *extensionsService) Enable(ctx context.Context, name string, req contract.EnableExtensionRequest) (contract.ExtensionEnableResult, error)
```

The `extension.enabled` event's `automation_started_count` equals `len(AutomationStarted)` by construction (same computed set).

### Data Models

New/changed storage, global catalog stream (`compozy.db`), per `eng-schema-migration` (declarative source + appended Goose SQL + `atlas.sum` + sqlc regen):

**Migration `00029` (additive — ships with phases B/C):**

| Column | Type | Rationale (purpose + shape) |
| --- | --- | --- |
| `extensions.network_requirement_digest` | `TEXT NOT NULL DEFAULT ''` | Confirmed digest of the manifest's `network_participation` block; `''` = never confirmed. Compared against the freshly computed manifest digest — equality means consent is current (ADR-005). |
| `extensions.network_confirmed_by` | `TEXT` | Real consent actor (R1 B-006): `operator` on operator transports, `agent:<session-scoped id>` via the native tool's canonical dispatcher — never a fixed literal; NULL until confirmed. |
| `extensions.network_confirmed_at` | `TEXT` | RFC3339Nano consent timestamp; NULL until confirmed. Pair-required with `confirmed_by` (validated in Go, as the activation spec did). |
| `extension_dev_links.network_requirement_digest` | `TEXT NOT NULL DEFAULT ''` | Dev-instance consent state (R1 B-001): a dev-linked instance — possibly with no `extensions` row at all — persists its own confirmed digest here; the reload gate reads it. |
| `extension_dev_links.network_confirmed_by` | `TEXT` | Dev-instance consent actor; same value rules as the published column. |
| `extension_dev_links.network_confirmed_at` | `TEXT` | Dev-instance consent timestamp; pair-required with its `confirmed_by`. |
| `extension_env_bindings.extension_name` | `TEXT NOT NULL` | Binding owner; `PRIMARY KEY (extension_name, workspace_id, env_name)`. |
| `extension_env_bindings.workspace_id` | `TEXT NOT NULL DEFAULT ''` | Instance scope (R1 B-001): `''` = the global published instance; a dev instance's rows carry its workspace. Bound server-side from caller scope, never client-supplied — cross-workspace reads/injection are structurally impossible (L-033). |
| `extension_env_bindings.env_name` | `TEXT NOT NULL` | Declared `requires_env` name the binding satisfies. |
| `extension_env_bindings.secret_ref` | `TEXT NOT NULL` | `vault:extensions/...` (or operator-supplied existing ref); never a value. |
| `extension_env_bindings.kind` | `TEXT NOT NULL` | Ownership tag (`extension_env`) so superseded-ref GC never deletes a foreign secret (MCP pattern). |
| `extension_env_bindings.created_at` / `updated_at` | `TEXT NOT NULL` | Audit/display. |

**Migration `00030` (cleanup — ships with phase D, after the projector code is deleted):** `DELETE FROM resource_records WHERE kind IN ('bundle','bundle.activation'); DELETE FROM resource_records WHERE owner_kind = 'bundle.activation';` — the second statement removes the materialized agent/soul/heartbeat/job/trigger/bridge/layout records that only the deleted projector would ever sweep. One-shot data cleanup in a numbered migration; no boot repair (L-008). No bundle-dedicated tables exist, so nothing else changes.

**Side-table-vs-JSON decisions:** `extension_env_bindings` is a **typed side table** — bindings are matchable state (looked up per (extension, workspace, env) at spawn, listed per instance, deleted individually); a JSON bag on the `extensions` row would recreate the forbidden pattern. Network confirmation is **columns on the instance's row** — `extensions` for the published instance, `extension_dev_links` for dev instances — 1:1 lifecycle state read on every gate, not opaque metadata. Inventory and preview are **read models** (no storage). Kit records reuse `resource_records` with `Owner{Kind:"extension"}` — no new table; the existing `(owner_kind, owner_id, kind)` index serves the owner queries. Automation overlays stay in their existing side tables.

**Publisher owner contract (R1 B-004)** — "records gain an owner" must be executable through the substrate, not asserted: `resources.Draft` gains an explicit optional `Owner *ResourceOwner` that the kernel persists when set (actor-derived ownership remains the fallback for all other callers); every extension publisher sets it via one shared constructor (`extensionOwner(name)`), keeping the existing daemon sync actor as the mutation source. Managed-record equality (`sameManagedRawRecord` and per-kind peers) compares **normalized owner and source in addition to scope + encoded spec**, so a pre-existing daemon-owned row with an unchanged spec is *not* considered current and is rewritten with the extension owner on the first reconcile after upgrade — convergence is proven by a seeded pre-cut-record integration test per kind (IT-017), not assumed.

### API Endpoints

Deleted (HTTP `internal/api/httpapi/routes.go:345-355`, UDS `internal/api/udsapi/routes.go:388-398`, spec `internal/api/spec/registry_bundles.go`, contract `internal/api/contract/bundles.go`):

| Method | Path |
| --- | --- |
| GET | `/api/bundles/catalog` |
| POST | `/api/bundles/preview` |
| GET / POST | `/api/bundles/activations` |
| GET / PATCH / DELETE | `/api/bundles/activations/{id}` |
| GET | `/api/bundles/network/settings` |

Added (both transports, `internal/api/core` shared handlers; spec operations + tags registered; OpenAPI + generated TS + E2E mocks co-ship per L-007):

| Method | Path | Description |
| --- | --- | --- |
| GET | `/api/extensions/{name}/inventory` | Shipped-vs-live kit items (unprivileged read; 404 unknown extension) |
| GET | `/api/extensions/{name}/preview` | Enable dry-run: publish set, conflicts, missing/unbound env, automation starting, network digest (unprivileged read) |
| GET | `/api/extensions/{name}/secrets` | `{declared_env: [...], bound_env_keys: [...]}` — presence only, never values |
| PUT | `/api/extensions/{name}/secrets` | Privileged; body `SetExtensionSecretsRequest` (write-only values); 400 undeclared name / dangling ref; 404 unknown extension |
| DELETE | `/api/extensions/{name}/secrets/{env_name}` | Privileged; unbind one name (secret GC per ownership kind) |

Changed: `POST /api/extensions/{name}/enable` accepts `{confirm_network_digest?: string}` (the exact digest being consented to — a boolean cannot ratify a digest the caller never saw, R1 B-006) and returns `ExtensionEnableResult` (R1 B-005); the single-update request gains `confirm_network_digest` while batch update (`--all`) accepts none and refuses digest-changing items per-item with their digests in the partial progress; `GET/PUT/DELETE .../secrets` operate on the caller-bound instance (operator transports: the global instance by default, the workspace instance for dev flows; agent callers: trusted session scope — `workspace_id` is never a request field); `ExtensionPayload` drops `bundles` (`ExtensionBundleSummaryPayload` deleted) and gains `bound_env_keys []string`, `network_requirement_digest string`, `network_confirmation_required bool`; skill payloads drop `installed_from_bundle`. Deterministic errors: enable/update/reload without valid confirmation → 409 `extension_network_confirmation_required` (payload carries the **current** digest — also the answer to a stale `confirm_network_digest`); undeclared binding name → 400 listing declared names; dangling ref → 400; same env name in both value and ref form → 400; agent-name conflict at enable → 409 `extension_agent_conflict` naming the colliding agents; vault sentinels keep their existing mappings. The diagnostics code `extension_blocked_by_bundle` is deleted from `internal/diagnosticcontract` + `internal/api/contract/diagnostics.go` (support-bundle diagnostic codes stay).

## Integration Points

None new. The program touches no external service: vault is local (AEAD, `compozy.db`), automation/window-manager/network are in-process domains, and the deleted surfaces had no external dependencies. GitHub/git distribution paths are untouched (checked: `internal/registry/**` has no bundle coupling).

## Delete Targets

Everything below disappears in this program, in the same change as its replacement (no fallback, no compat shim, no placeholder, no `@deprecated` stub). Verified against the worktree; ⚠ marks shared files that are **stripped, not deleted**. Homonyms that MUST survive: support diagnostics bundles (`internal/support/**`, `/api/support/bundles*`, `compozy support bundle`, `internal/daemon/support_bundle_status.go`, web `systems/support/**`, site `operations/support-bundles.mdx`, diagnostics codes `CodeBundleConsentRequired|PartialFailure|SizeExceeded`), skills `SourceBundled`/`--source bundled` (`internal/extension/registry_bundled_cache.go`, `internal/hooks/types.go:41`, `internal/codegen/sdkts/generate_maps.go:141`), task ContextBundle, `crash_bundle_path`, CLI `outputBundle` helpers, sigstore/JS bundling, the extension archive/dev-lane "bundle generation" vocabulary (`BuildBundle`, `bundle_generation` column, `extensionBundle` CLI helpers), and `docs/qa/scenarios/ET-dev-cycle-skill-bundle.md`.

**Go runtime**

- `internal/bundles/**` — entire package (33 files incl. `model/`), all 9 test files.
- `internal/extension`: `bundle.go`, `bundle_agent_loader.go` (rehomed, ADR-007), `bundle_bridge_validation.go`, `bundle_decode.go`, `bundle_files.go`, `bundle_job.go` (rehomed, ADR-002), `bundle_layout_clone.go`, `bundle_layout_loader.go` (rehomed), `bundle_participation.go`, `bundle_path_security.go` (rehomed as generic resource path guard if still needed by the new loaders), `bundle_profile_validation.go`, `validate_bundle.go` (the bundle-spec name-collision validator — the archive `ValidateBundle` in `build.go` stays), `bundle_additional_test.go`, `registry_bundles_test.go`; `Bundles` field in `manifest.go:83` + `manifest_normalize.go:126`; `loadBundleResources` + `ext.bundles` in `manager_supervision.go`/`manager.go:167,187`/`manager_resource_loading.go:176-181`; `cloneBundleSpecs`/`cloneBundleAgents`/`cloneBundleLayouts`/`normalizeBundleChannels`/`normalizeBundleBridges` in `manager_snapshots.go` (⚠ `cloneBundleTaskConfig` is used by `host_api_workspace_automation_mapping.go:113,139` — rename/relocate as an automation task-config clone helper); `bundleSummaryPayloads` + `Bundles` projection in `describe.go:79,133-147`; surface families `FamilyBundles`/`FamilyBundleActivations` + both kind descriptors in `surfaces/registry.go:23,26,93-94,111-112`; `ErrExtensionHasActiveBundles` + `ensureNoActiveBundles` + the direct `resource_records` SQL scan (`registry.go:20-22,99`, `registry_lifecycle.go:22,54-105`).
- `internal/daemon`: `bootBundles` (`boot_automation_bundles.go:90-132`; file re-filed by responsibility), `bundle_resources.go`, `bundle_resources_projection.go`, `bundle_resources_store.go`, `bundle_resources_test.go`, `native_bundle_resource_tools.go`; state/deps/wiring: `boot.go:121`, `boot_components.go:23`, `boot_resource_graph.go:102,202`, `daemon.go:162`, `runtime_deps.go:77`, `server_options.go:31,102`, `extensions.go:312,388-389`, `extensions_service.go:24,67`, `daemon_resource_projection.go:126-127`, `native_tools.go:191-195`, `native_tool_availability.go:43,128,130`, `native_tool_bindings.go:51`, `native_tool_dependencies.go:85,87`, `native_tools_dependencies_builder.go:74,76-78`, `native_marketplace_tools.go:91`, `native_extension_tool_errors.go:27`, `native_tool_http_errors.go:10`; ⚠ `agent_skill_catalog.go:171` (`PackageOwned` comparison replaced per ADR-004).
- `internal/api`: `contract/bundles.go` (20 types), `contract/extensions.go:127,134-135` (`Bundles` field + `ExtensionBundleSummaryPayload`), `contract/marketplace.go:9,34,114-138` (`MarketplaceKindBundle`, `MarketplaceBundleProfilePayload`, `MarketplaceBundleDetailPayload`, `Bundle` entry field), `contract/diagnostics.go:70` + `internal/diagnosticcontract/diagnostics.go:83,210` (`extension_blocked_by_bundle`); `core/bundles.go`, `core/marketplace_bundles.go`, `core/bundle_category_payload_test.go`; ⚠ strips in `core/{interfaces.go:11,164-175, base_handlers.go:54,132,220, resources.go:148-151, marketplace.go, marketplace_detail.go, marketplace_list.go, marketplace_routes.go:7-8, handlers_observability.go:144-151, extensions.go:319}`; `spec/registry_bundles.go` + `spec/operation_registry.go:18` + `spec.go:32,97,148,247` + `marketplace_parameters.go:11` + the `--installed-name` bundle copy in `marketplace_entry_operation.go:18`; route blocks `httpapi/routes.go:42,345-355` + `udsapi/routes.go:34,388-398` + their option/plumbing lines (`httpapi/server.go:60,136-140`, `server_handler_config.go:27`, `handlers.go:40,149`; `udsapi/server.go:76,157-161`, `handler_config.go:35`, `server_handler_config.go:21`, `server_handlers.go:45`).
- `internal/cli`: `bundle.go`, `bundle_activation_flags.go`, `bundle_output.go`, `bundle_rows.go`, `bundle_tables.go`, `bundle_update.go`, `bundle_test.go`; `newBundleCommand` registration (`root.go:142`); ⚠ bundle client methods + record types in `client_extensions_bridges.go:157-240` and the fakes in `helpers_test.go:107-111`.
- `internal/tools` + `internal/toolmeta`: tool IDs + toolset (`builtin_ids.go:347-356,466-467`), toolset row (`builtin/toolsets.go:206`), ⚠ `builtin/bundles_resources.go` (split: `compozy__resources_*` descriptors move to their own file; bundle descriptors + `bundles.read`/`bundles.write` capability gates delete), registration (`builtin/descriptors.go:47`), toolmeta rows (`native_entries.go:31-35`), golden fixture rows (`builtin/testdata/native-tool-catalog.json:354-430`), assertions (`builtin/builtin_test.go:1105-1106`, `toolmeta/registry_test.go`).
- `internal/resources`: `projector.go:10-11,29-31,129-180` (`bundleKind`/`bundleActivationKind`, `BundleActivationProjector`, `NewBundleActivationProjectorRegistration` + validation — the whole mixed-kind escape hatch); ⚠ tests using `"bundle"`/`"bundle.activation"` as fixture kinds (`reconcile_test.go:543-659`, `reconcile_integration_test.go:46-73`, `kernel_test.go:180,207`, `typed_test.go`, `typed_integration_test.go`, `perf_bench_test.go:129-159`) are **renamed to neutral fixture kinds**, not deleted.
- `internal/store/globaldb`: `global_db_bundles.go` + test, `ListBundleActivationResourceSpecs` (`queries/global_small.sql:17` + sqlc regen), `BundleRepo` registration in `repositories.go`.
- `internal/skills`: `InstalledFromBundle` (`types.go:33`, `resource_spec.go:37,65,93,128`, writer `registry_resource_projection.go:85-86`), contract `skill_payloads.go:90`, conversion `conversions_skills.go:33`, CLI `skill_workspace.go:144` + `skill_output.go:249-250`.
- `magefiles/boundaries.go:156` (marketplace ↛ bundles rule).
- ⚠ Shared test suites (strip cases, keep suites): `internal/api/core/{extensions_test.go:492, payload_helpers_test.go:240, marketplace_test.go, network_test.go:24,693-712, resources_test.go}`, `internal/api/httpapi/{handlers_test.go, httpapi_integration_test.go, resources_test.go, transport_parity_integration_test.go:623-629}`, `internal/api/udsapi` mirrors, `internal/daemon/{agent_skill_resources_test.go, native_marketplace_tools_test.go, native_tools_test.go}`, `internal/extension/manager_test.go`.

**Generated artifacts (regenerate clean via `make codegen`)**: `openapi/compozy.json` (tag `bundles`, 8 ops, 5 paths; skill payload field), `web/src/generated/compozy-openapi.d.ts`, `web/src/routeTree.gen.ts`, `internal/store/globaldb/sqlcgen/**`, `atlas.sum` refresh, native-tool catalog golden, generated CLI docs.

**Web** — delete whole files: routes `_app/marketplace.bundles.tsx` + `_app/marketplace.bundles.activations.$id.tsx`; `systems/extensions/components/bundle-activation-detail.tsx` (+test) + `hooks/use-bundle-activation-detail.ts`; `systems/marketplace/components/{bundle-activation-dialog.tsx, bundle-activation-dialog-store.ts (+test), bundle-activation-model.ts, use-bundle-activation-dialog.ts}`; `systems/marketplace/routes/bundle-activation-detail.stories.tsx`. ⚠ Strip shared: `systems/marketplace/{types.ts (kind unions + 4 request/response types + union arms), lib/marketplace-kind-config.ts:56-66, lib/query-keys.ts:46, lib/marketplace-installed-items.ts:139, components/index.ts:1-3, adapters/marketplace-actions-api.ts:102,118, adapters/marketplace-api.ts, mocks/*, 13 shared components incl. marketplace-detail-skill-manage.tsx:175, hooks/use-marketplace-actions.ts + use-marketplace-kind-page.ts, stories}`; `systems/extensions/{index.ts (8 barrel exports), types.ts:8-12, lib/query-keys.ts:21-22, lib/query-options.ts:38-48, adapters/extensions-api.ts:190-234, components/extension-dialogs.tsx, hooks/*, mocks/*}`; `systems/os/apps/marketplace/{marketplace-window.tsx:1,28-29, marketplace-detail-location.tsx:46}`; `systems/session/lib/tool-labels.ts:71` (`bundles: Boxes` icon row); `storybook/route-story-registry.ts:110-113,278-280`; Vitest strips across the ~20 files listed in the inventory; Playwright: the bundle journey + helpers + parity walk in `web/e2e/__tests__/marketplace.spec.ts` and the 3 bundle test IDs in `web/e2e/fixtures/selectors.ts` + `scenario-contracts.ts:265` (⚠ `bundled-extension-seeder` and `runtime-seed.ts` SourceBundled seeding are homonyms — keep).

**Site docs** — delete: `content/runtime/core/resources/bundles.mdx`, `content/runtime/api-reference/bundles.mdx`, `content/runtime/cli-reference/bundle/**` (9 files + meta). ⚠ Strip sections: `core/extensions/{install.mdx, develop.mdx (rewrites to teach the new kit path), manifest.mdx (resources.bundles row + network-participation block rewrite to lifecycle confirm), index.mdx, permissions.mdx (reword archive homonym)}`, `core/marketplace/index.mdx` (incl. the whole "Preview And Activate A Bundle" section), `core/resources/index.mdx` (kind rows/examples), `core/tools/{toolsets.mdx:68, index.mdx:134}`, `api-reference/index.mdx:45`, `core/skills/bundled.mdx:18,37` (homonym file — 2 lines only), `core/bridges/{index.mdx:213, operations.mdx:697, adding-a-bridge.mdx:792}`, `cli-reference/{compozy.mdx, index.mdx, marketplace/search.mdx}`, `core/configuration/config-toml.mdx:874`, root `README.md:48,160,181`. Nav: `lib/runtime-navigation.ts:67`, `api-reference/meta.json:18`, `cli-reference/meta.json:22`, `core/resources/meta.json:4`. Site tests: `lib/__tests__/{runtime-docs-discovery.test.ts:78, runtime-tools-canonical-docs.test.ts:59,67,129, runtime-docs-truth.test.ts:199}` updated to the post-cut truth. **New docs**: static kit authoring (agent dirs + sidecars, automation TOML, layouts) in `develop.mdx` + manifest reference; secrets binding guide; inventory/preview + confirm in extension operations; CLI reference regen (new verbs/flags, `bundle` group gone).

**Skills / QA / ecosystem / design / memory**

- `skills/compozy/`: `references/capabilities-and-bundles.md` → renamed `capabilities.md` with the "Bundles" section deleted; `native-tools.md` (heading, `compozy__bundles_*` rows, cross-refs), `tools-and-skills.md` (kind lists, installed-from wording), `extension-authoring.md` (`resources.*` list + cross-ref; gains agents/automation/layouts/secrets/confirm authoring), SKILL.md router row updated. Prompt-rune budget note: the startup prompt sits at 31,978/32,000 — bundle removals free runes before kit additions land; the net must stay ≤ 32,000 (tested by the existing budget gate).
- QA scenarios — delete: `ET-024..ET-030`, `NB-023`, `ET-web-bundle-activation-detail`, `ET-web-bundle-preview-activate`. Strip: `ET-020:23`, `ET-033:23`, `ET-cli-marketplace-refresh:7`, `RT-reserved-builtin-agent-names:8,23`, plus the grep-positive set to verify at execution (`ET-003/004/016/035/049/053`, `ET-web-*`, `MS-*`, `NB-025`, `RT-028`). Journeys/charters: `J-agent-marketplace-parity:20,47`, `J-marketplace-acquisition:69`, `CH-agent-marketplace-parity:24`, `CH-network-admin-lifecycle:20`, `CH-reserved-builtin-name-sweep:30` + the verify list. Seeds: `_seeds/final-qa/_children/08-extensions-bridges.md` (EXT-02 scenario + CLI/invariant tables), `11-api-cli-parity.md:48,358`, `_master-qa-plan.md:78`, `_seeds/feature-stories/03_analysis_extensibility-tools.md:77-83`. New `untested` scenarios: kit enable (sidecars/automation/layouts), secrets binding, inventory, preview, network confirm (enable + update refusal). Historical bug files stay annotated (content-addressed registry is append-only).
- Ecosystem: `docs/ecosystem/bundle-opportunities.md` archived/deleted; `docs/ecosystem/README.md` bundle pillar rewritten; `extension-opportunities.md` reframed. Design: `docs/design/opendesign/_done/marketplace/bundle-activation-detail.html` (+artifact) deleted; `marketplace-detail.html` prototype stripped of the bundles kind.
- Memory/instruction checklists: `CLAUDE.md:28,49,57` (impact-audit template "bundles" entries), `internal/CLAUDE.md:66,91`, `AGENTS.md`/`internal/AGENTS.md` living bundle lines, `docs/_memory/standing_directives.md:177` (SD-011 surface list), `docs/_memory/spec-authoring-playbook.md:74,153`, `skills/compozy` + `cy-web-docs-impact`/`cy-spec-preflight` skill trigger lists mentioning bundles as a living surface. Historical lessons/analyses stay.
- `.compozy/tasks/marketplace/**` and product-bundle legs in other local task trees: annotate as pre-hard-cut (skeeper-managed, not in git).
- Doc-comment rewording (not deletion): `internal/automation/types.go:27`, `internal/automation/model/types.go:32`, `internal/bridges/types.go:67`, `internal/api/contract/extension_authoring.go`, `internal/extension/contract/authoring.go` ("bundle" → archive/package wording).

## Safety Invariants

1. **Install is inert; enable is the only kit switch.** Installing/linking never publishes records, starts automation, spawns subprocesses beyond describe-free install paths, or binds anything. Static resources publish only for `Enabled && Registered` extensions (existing gate, now covering all kinds).
2. **Enable output is the consent record for automation** (ADR-002, R1 B-005): the enable **action result** (`ExtensionEnableResult.AutomationStarted`) enumerates exactly the effectively-enabled definitions that became runnable in the committed operation — name-sorted, identical across CLI/HTTP/UDS/native output, equal to the `extension.enabled` event count, and never mirrored into status/list payloads; preview shows the same list before the act.
3. **Package automation is read-only**: records carry `Source: package` + `Owner{extension}`; create/update/delete through the automation API stays `ErrDefinitionReadOnly`; only `enabled_override` overlays mutate effective enablement; overlays are garbage-collected when their package definition disappears (disable/remove/update-drop) — no orphan overlay rows.
4. **Owner attribution is universal and convergent** (R1 B-004): every record published from an extension package carries `Owner{Kind:"extension", ID:<name>}` set explicitly on the draft (never inferred from the sync actor); managed-record equality includes normalized owner + source, so pre-existing unowned rows rewrite on the first reconcile (seeded-row test per kind); `PackageOwned == (Owner.Kind == "extension")`; sidecar↔agent matching requires same owner+scope; disable/remove deletes exactly the owned/sourced set via declarative sync.
5. **Secret values never cross surfaces**: binding requests are write-only; values enter via hidden TTY/stdin (never argv); presence projections expose names only; every resolved value registers for redaction before injection; no secret value appears in payloads, errors, events, SSE, logs, or tool transcripts (leak-asserted in tests).
6. **Binding writes are transactional and ordered** (R1 N-006): each env name carries exactly one of value/ref (overlap → 400); mutation order is sorted normalized env names; failure rolls back in reverse restoring prior value+kind per ref (detached timeout context per L-001); superseded-ref GC deletes only refs tagged `extension_env` and not referenced elsewhere; bind-to-ref requires namespace match + `metadata.Present` — a dangling ref is a 400, never a deferred spawn failure.
7. **Spawn env resolution is instance-exact and declaration-intersected** (R1 B-001/B-003): order is allowlist → manifest `env` → manifest `secret_env` → the launching instance's own `(name, workspace_id)` bindings — no cross-scope fallback in either direction; injection intersects bindings with the current normalized `requires_env` (stale bindings are listed, never injected); resolution failure aborts launch naming key+ref; `missing_env` = declared − (process env ∪ non-stale bound); enable never blocks on missing env — it reports and suggests the bind command.
8. **Network consent is digest-exact, actor-true, and instance-scoped** (ADR-005, R1 B-001/B-006): confirmation requires the expected digest — the coordinator re-reads the candidate manifest under the lifecycle lock and rejects a mismatch with the current digest (a stale digest is never ratified); the recorded actor is the real caller identity (`operator` or `agent:<id>`); published state lives on `extensions`, dev-instance state on `extension_dev_links`; only a digest change invalidates confirmation; `update` compares the staged digest **before any swap** and a refused update leaves zero changed state; batch update refuses digest-changing items per-item and never blanket-confirms.
9. **Preview never mutates**: no records written, no vault writes, no subprocess start, no confirmation recorded; preview and enable derive from the same desired-state computation (equivalence-tested).
10. **Agent-name conflicts fail closed**: a shipped agent name colliding with a visible non-owned agent (global catalog + reserved builtin names) fails enable deterministically (409 naming the agents) and surfaces in preview first; publish never silently overwrites a non-owned record.
11. **Disable/remove gate only on extension state**: `ErrExtensionHasActiveBundles` and the activation scan are deleted; no replacement gate reads `resource_records` directly from the registry.
12. **Orphan cleanup is a numbered migration** (`00030`), shipped with the code cut; no `EnsureSchema`, no boot repair; migration bytes append-only; fresh/reopen/ahead/integrity/equivalence suites extended.
13. **Homonym surfaces are untouched**: support bundles, `SourceBundled`, ContextBundle, `crash_bundle_path`, `outputBundle`, sigstore/JS bundling, and dev-lane generation vocabulary keep byte-identical behavior (smoke-tested).
14. **New native tools are read-only** (`extensions_inventory`, `extensions_preview`); no native tool writes secrets (MCP parity); `extensions_enable` carries the confirm field but stays gated by its existing risk class.
15. **Generated contracts stay byte-checked**: describe/`resources` schema, OpenAPI, TS types, sqlc, native-tool catalog golden, and CLI docs regenerate in the same changes; `make codegen-check` is the drift gate.
16. **One lifecycle authority per extension name** (R1 B-002): every lifecycle mutation (enable, disable, install, update, remove, dev link/unlink, reload) runs inside the daemon's per-name lifecycle coordinator — preflight fence (manifest digest + registry version), staged mutation, confirmation write, reload, reconcile, and response construction serialize as one operation with a defined commit point; reverse rollback restores registry row, files, confirmation tuple, enabled bit, and the last-good running generation; a refused/failed operation releases with byte-identical persisted and running state; no peer path mutates registry, files, or confirmation outside the coordinator.
17. **Bindings and confirmations never cross instances** (R1 B-001, L-033): binding rows and confirmation state are keyed by `(extension_name, workspace_id)` with server-side workspace binding only; a workspace instance cannot read, inject, or confirm on behalf of the global instance or another workspace, and vice versa — asserted by cross-workspace non-leakage tests.

## Extensibility Integration Plan

- **Extension manifests**: `resources.bundles` deleted; `resources.automation` + `resources.layouts` added; `resources.agents` semantics hard-cut to dir-per-agent (ADR-007); `network_participation` shape unchanged but now enforced on the lifecycle (ADR-005). In-repo manifests: `extensions/dev-cycle` (already dir-per-agent) revalidates; `extensions/bridges/*` unaffected (no static kit resources).
- **Describe/SDK contracts**: `DescribePayload.resources` added and generated into both SDKs (ADR-008); conformance fixtures extended; no protocol method changes.
- **Hooks**: no impact — checked `internal/hooks` (no bundle hook events; `HookSkillSourceBundled` is a homonym); hook source/priority tiers unchanged.
- **Skills / capabilities**: `installed_from_bundle` provenance deleted; skill loading/`SourceBundled` untouched; glossary updated (bundle entries removed; kit vocabulary added).
- **Tools / resources**: −`compozy__bundles_{list,info,activate,deactivate,status}`, −toolset `compozy__bundles`, −capabilities `bundles.read`/`bundles.write`; +`compozy__extensions_inventory`/`compozy__extensions_preview` (read-only); `compozy__extensions_enable` input schema gains `confirm_network_digest`; surfaces registry loses the two bundle families; resource kinds `bundle`/`bundle.activation` and the mixed-kind projector seam deleted.
- **Bundles**: the surface itself is the delete target (this program).
- **Registries / marketplace**: kind set becomes `{extension, skill, mcp}`; curated feed drops bundle rows; distribution (`internal/registry/**`) untouched.
- **Bridge SDKs / bridges**: bridge adapter path untouched; bundle bridge presets die without replacement — bridge instances remain the separate `compozy bridge setup` operator step with `bridge_secret_bindings`; `BridgeInstanceSourcePackage` comment reworded. The likely-defect where `UpdateInstanceState` can enable a package bridge that reconcile then reverts disappears with the only package-bridge producer.
- **MCP sidecars**: no impact — checked `internal/mcp` + manifest `resources.mcp_servers` (unchanged); the MCP secrets machinery is mirrored, not modified.
- **Protocol docs**: extension lifecycle docs gain kit/consent semantics; network protocol docs unaffected (declared-channel projection is deleted, not moved).

## Agent Manageability Plan

- **CLI verbs** (all with `-o json|jsonl|toon`): new `compozy extension inventory <name>`, `extension preview <name>`, `extension secrets set|bind|list|unset` (set = value via hidden TTY/stdin; one form per env name); changed `extension enable [--confirm-network-requirement <digest>]`, `extension update [--confirm-network-requirement <digest>]` (single verb only — `--all` refuses digest-changing items per-item with their digests), enable success output rendering `ExtensionEnableResult` (started automation enumerated) + next-step hints (`secrets set` suggestion on missing env); deleted `compozy bundle *` (whole group).
- **HTTP/UDS parity**: every new/changed surface registers in both `internal/api/httpapi` and `internal/api/udsapi` (table above); reads unprivileged, mutations privileged on HTTP.
- **Native tools**: +2 read-only tools with structured outputs identical to the routes; enable tool input extended; −5 bundle tools; toolset description for `compozy__extensions` updated to actual membership.
- **Deterministic errors**: `extension_network_confirmation_required` (409, always carries the **current** digest — the remediation for both a missing and a stale `confirm_network_digest`), `extension_agent_conflict` (409, names agents), undeclared-binding-name (400, lists declared names), dangling-ref (400), value+ref overlap for one env name (400), vault sentinel mappings unchanged; every new diagnostic renders through `renderHumanExecutionError` and structured output unchanged.
- **Status/config discovery**: `extension status` keeps health focus and gains nothing (ADR-006); doctor extension probe extends `missing_env` to "unbound" with the exact `compozy extension secrets set <name> --env <KEY>` suggested command; `compozy config set` registry unchanged (no new keys).
- **Docs for the agent path**: `skills/compozy` extension-authoring + operations references updated (see Web/Docs Impact).

## Config Lifecycle

**No `config.toml` keys are added, changed, or removed.** Checked surfaces: the `[extensions.*]` registry (`trust`, `sources`, `dev`, `resources` — `internal/config/config_extensions_sandbox.go:20-25` + the ext-improvs ADR-007 key set) is untouched; no bundle config key ever existed (verified: `internal/config` grep yields only sigstore/task-context homonyms); vault keying (`COMPOZY_VAULT_KEY` / `vault.key`) unchanged; automation `[automation]` config-source jobs unchanged. Bindings and confirmations are runtime state (SQLite), deliberately not config (ADR-003 alternative 2 rejected). Site `config-toml.mdx` changes only where prose references bundle materialization (`:874`).

## Web/Docs Impact

- **`web/src/systems/marketplace`**: kind system collapses to `{extension, skill, mcp}` (`types.ts`, `marketplace-kind-config.ts`, kind pages/cards/dialogs/hooks/mocks per Delete Targets); shared shells (MCP/extension/skill flows) must keep working — asserted by the surviving Vitest + Playwright coverage.
- **`web/src/systems/extensions`**: bundle activation detail/hooks/dialogs deleted; extension detail gains a **kit inventory panel** consuming `GET /api/extensions/{name}/inventory` (shipped vs live per kind) and surfaces `bound_env_keys` + network-confirm state from the extended payload; enable flow gains the confirm affordance when `network_confirmation_required`, and the **existing update affordance** (`use-extension-detail-state.ts` → `marketplace-detail-extension-manage.tsx`, shipped by ext-improvs) handles the new pre-swap 409: it surfaces the returned digest and retries the update mutation with `confirm_network_digest` on operator approval — the enable and update flows share one confirm affordance (R1 N-005). Truthful UI: every control maps to a shipped route above; nothing else is invented.
- **`web/src/systems/os`**: marketplace window activation branch for bundle routes removed.
- **Generated**: `make codegen` refreshes `openapi/compozy.json` + `web/src/generated/compozy-openapi.d.ts` + `routeTree.gen.ts`; MSW fixtures and E2E mocks/matchers co-ship (L-007); Playwright bundle journey replaced by an extension-kit journey (install → secrets → enable+confirm → inventory → disable).
- **`packages/site`**: deletions/strips/new pages per Delete Targets; `develop.mdx` + `manifest.mdx` teach the kit authoring (agent dirs + sidecars, automation TOML, layout JSON, describe `resources`); new secrets-binding guide; extension operations page gains inventory/preview/confirm; CLI reference regen (`bundle` group gone, new verbs/flags); nav metas updated; the three site test files updated to post-cut truth.
- **`skills/compozy`**: rename + rewrite per Delete Targets; router table updated; rune budget respected.
- **Root docs**: `README.md` product language; `CLAUDE.md`/`AGENTS.md`/`internal/CLAUDE.md`/`internal/AGENTS.md` checklist lines; `docs/_memory` standing-directive/playbook surface lists.
- **QA tracker impact**: scenarios deleted (`ET-024..030`, `NB-023`, `ET-web-bundle-*`), stripped (`ET-020`, `ET-033`, `ET-cli-marketplace-refresh`, `RT-reserved-builtin-agent-names`, + verify list), and **new `untested` content-addressed scenarios**: `ET-ext-kit-enable` (agents+sidecars/automation/layouts live on enable, output enumerates automation), `ET-ext-secrets-binding`, `ET-ext-inventory`, `ET-ext-preview`, `ET-ext-network-confirm` (enable gate + update refusal), `ET-web-extension-kit-inventory`. Flag, don't retest.

## Impact Analysis

| Component | Impact | Description / Risk | Action |
| --- | --- | --- | --- |
| `internal/bundles` | **deleted** | whole package + 14 importers | phase D; imports removed with wiring |
| `internal/extension` | modified (heavy) | bundle schema out; loaders/bindings/confirm/inventory in — highest-risk surface | phases A–C + invariants 1–11 |
| `internal/extension/contract` + SDKs | modified | describe `resources` | codegen co-ship, byte-checked |
| `internal/daemon` | modified (heavy) | publishers + services + gates + tools; wiring only | composition-root discipline (SD-008) |
| `internal/api/*` | modified | −8 routes/ops, +5 routes, payload changes | contract co-ship (L-007) |
| `internal/cli` | modified | −1 command group, +6 verbs/flags, enable output | golden-output tests |
| `internal/tools`/`toolmeta` | modified | −5/+2 tools, toolset, capabilities | catalog golden + toolmeta tests |
| `internal/vault` | modified (light) | ref namer + ownership kind | mirror of `mcp_refs.go` + tests |
| `internal/store/globaldb` | migrations | `00029` additive, `00030` cleanup; sqlc regen | `eng-schema-migration` suites |
| `internal/resources` | modified | projector escape hatch deleted; fixture kinds renamed | reconcile suites stay green |
| `internal/skills` | modified (light) | `InstalledFromBundle` deleted end-to-end | contract regen |
| `internal/automation` | modified (light) | comments; overlay GC on package-definition removal | overlay GC tests |
| `web/` | modified | 11 deletes + ~40 strips + inventory panel + confirm affordance | turbo lane + Playwright |
| `packages/site` + `skills/compozy` + root docs | modified | deletes + rewrites + new guides | docs build + site tests |
| `docs/qa` | modified | −10 scenarios, strips, +6 untested | QA pair tail |
| `magefiles` | modified | boundaries rule removed | same-commit rule |

## Testing Approach

Strategy only — every concrete case with an ID lives in `_tests.md` (UT/IT/E2E), each mapped to a canonical owning suite per `eng-consolidate-test-suites`.

- **Unit** (`go test -race`, scoped; Vitest for web): new loaders (agent dirs/sidecars, automation TOML incl. webhook rejection, layouts) with error paths; ref namer; binding store + transactional service (rollback, GC, undeclared/dangling rejection); digest gate state machine; inventory/preview computation + equivalence; CLI rendering golden tests; error mappings; publisher owner attribution.
- **Integration** (`+integration`): migration suites (fresh/reopen/ahead/integrity/equivalence for `00029`+`00030`, incl. cleanup semantics on a seeded DB); enable→publish→disable→delete round trips per kind with owner assertions; overlay GC; scheduler start/stop of package jobs; secrets bind→spawn injection with real vault; confirm flows (enable 409, update pre-swap refusal per source, batch partial); HTTP/UDS transport parity for added/changed/deleted routes; native tools.
- **E2E runtime** (Go harness): the kit journey on a fixture extension (install → secrets set → enable with confirm → inventory → invoke → update-with-digest-change refusal → disable → gone); dev-cycle regression (enable exposes skills/loops/agents/tools; disable removes them); homonym smoke (support bundle create/status/download; `--source bundled` skills).
- **E2E web** (Playwright): marketplace without the bundle kind; extension detail inventory panel; enable-confirm affordance; the deleted bundle journey replaced.
- **Verification gates**: `make codegen && make codegen-check` after every contract phase; scoped `make lint` + package tests during phases; one full `make verify` at completion (includes site tests asserting post-cut nav/docs); grep gate for living product-bundle references (`/api/bundles`, `compozy bundle `, `compozy__bundles_`, `MarketplaceKindBundle`) run as a completion check, expecting only historical/homonym hits.

## Development Sequencing

### Build Order

1. **Phase A — Kit completeness on enable (R-P0-1..4)**: dir-per-agent loader + sidecar records + catalog matching (`PackageOwned` redefinition), owner attribution across all publishers, automation loader/provider + enable-as-consent output + overlay GC, layout loader/provider, agent-conflict validation, describe/SDK `resources` + build copying (ADR-008), fixture + dev-cycle updates. *Gate*: scoped extension/daemon suites + codegen-check.
2. **Phase B — Secrets binding (R-P0-5)**: migration `00029` (bindings table + confirm columns land together), vault ref namer, binding store/service, CLI verbs + routes + payload additions, spawn injection order, doctor probe extension. *Gate*: migration suites + bind→spawn integration + leak assertions.
3. **Phase C — Lifecycle confirm + inventory/preview (R-P1-1..4)**: enable/update/reload gates + batch partial reporting, inventory + preview services/verbs/routes/tools, preview-enable equivalence, events. *Gate*: confirm-flow integration per source + parity tests.
4. **Phase D — The hard cut (R-HC-1..7, R-P1-5)**: delete everything in Delete Targets (Go, contract, spec, routes, CLI, tools, resources seam, skills field, store, boundaries rule), migration `00030` cleanup, regenerate all artifacts, rename resource-fixture kinds, reword source comments. *Gate*: full scoped sweep + codegen-check + grep gate + homonym smoke.
5. **Phase E — Web (R-HC-1, R-AM-1)**: dedicated deletes, shared strips, kind collapse, inventory panel + confirm affordance, MSW/E2E updates, Playwright journey replacement. *Gate*: turbo web lane + Playwright.
6. **Phase F — Docs, skill, QA flags (R-HC-3, R-AM-2..4)**: site deletes/rewrites/new guides + nav + site tests, `skills/compozy` rewrite (rune budget), README/CLAUDE/AGENTS/memory checklists, ecosystem docs, QA scenario deletes/strips/new-untested, glossary. *Gate*: docs build + site tests + full `make verify`.
7. QA tail: `qa-report` + `qa-execution` close `_tasks.md` (added by `cy-create-tasks`).

### Technical Dependencies

A → B (bindings report against loaders' `requires_env` view) → C (preview shows unbound env; confirm columns from `00029`) → D (replacements live before the cut so dev-cycle/fixtures never lose a working kit path) → E (web needs the post-cut contract + new routes) → F (docs describe the final truth). No external dependencies; all migrations are local SQLite.

## Monitoring and Observability

Canonical extension events (append-only store) extended with the per-event required-key matrix pattern (`internal/events/extension.go` + `RequiredFields()`):

| Event | Required correlation keys |
| --- | --- |
| `extension.secrets.updated` | `extension_name`, `workspace_id` (empty for the global instance), `bound_count` (never names-with-values, never refs' secret content) |
| `extension.secrets.update_failed` | `extension_name`, `workspace_id` |
| `extension.network.confirmed` | `extension_name`, `workspace_id`, `digest`, `confirmed_by` |
| `extension.enabled` (existing) | gains `automation_started_count` (== `len(ExtensionEnableResult.AutomationStarted)`) |

Scope of the claim (R1 N-001): **success transitions plus the secrets failure event above emit canonical events; consent/validation refusals (409/400) are deterministic errors, not events.** The coverage-matrix test asserts exactly this set with exact keys at the emitting call sites; the same assertions verify no secret value or ref plaintext appears in any payload. `slog` fields follow the existing `extension`, `phase`, `reason_code` discipline. Metrics: bindings count, confirm refusals, kit publish counts per kind.

## Technical Considerations

### Key Decisions

Recorded as ADR-001..008. Additional in-spec decisions: **two migrations, not one** — additive schema ships with the features (B), destructive cleanup ships with the cut (D), because running the cleanup while the projector still exists would let enabled extensions repopulate the deleted rows; **automation trigger `event = "webhook"` stays rejected** for package triggers (carried from bundles — webhook secrets/slug lifecycle is operator-owned); **package automation targets only agents shipped by the same extension** (R1 N-003 — a kit is self-contained; targeting an external visible agent would create a dependency that can disappear independently, and operator-authored dynamic jobs already cover that case); **no cross-scope binding fallback** — a dev instance never inherits global bindings nor vice versa (isolation beats convenience); **stale bindings are listed, never injected, and removed only by explicit `secrets unset`** (auto-delete on update would lose operator data on a transiently dropped declaration and complicate update rollback); **local-path installs have no update lifecycle** (R1 B-007 — `UpdateMarketplaceManaged` requires a registry identity; the refresh path for a local install is reinstall, and workspace development uses the dev-link/reload lane); **no declared-channel projection** on extensions (the bundle `network/settings` read model dies without replacement — the manifest requirement + consent record is the whole network story this program); **`resources` test fixture kinds renamed, not deleted** — the reconcile/kernel suites keep their coverage with neutral kind names.

### Known Risks

- **Full-reload blast radius**: enable/disable/update still tear down and restart every extension (`Manager.Reload`); the new kinds ride the existing behavior — unchanged, but kit-bearing extensions make reloads heavier. Accepted; an incremental-reconcile program is future work.
- **Preview/enable divergence** — mitigated by the shared-computation design + equivalence test (invariant 9).
- **Missed living bundle references** — mitigated by the verified inventory (this spec) + the grep gate + site/toolmeta/catalog golden tests.
- **Official-skill rune budget** — deletions land before additions in the same task; the budget gate enforces ≤ 32,000.
- **Shared-file strips in web** — the marketplace kind union collapse touches ~40 files; typecheck + the surviving kind pages' tests are the net.

## Assumptions / Defaults

- Baseline is the `ext-improvs` branch state; this program starts after its QA execution completes.
- Kit scope is global-only; workspace-scoped enable is future work (ADR-004).
- Package automation defaults `enabled = true`; enable is the consent act; install is inert (ADR-002).
- Bindings are instance-keyed (`(extension, workspace_id, env)`, `''` = global) with no cross-scope fallback; operator bindings override authored `secret_env` for the same name; undeclared names are rejected; no native secret-write tool (ADR-003).
- Confirmation records the real caller identity (`operator` or `agent:<id>`) and always the exact expected digest; published-instance state lives on `extensions`, dev-instance state on `extension_dev_links`; dev-lane reloads honor the same digest gate (ADR-005).
- Inventory/preview are unprivileged reads; preview of an enabled extension reports the reload delta (ADR-006).
- Alpha data policy: hard cuts with numbered migrations; orphaned bundle rows deleted by `00030`; no compat shims, aliases, or dual fields anywhere.
- Bridge presets, profiles, and workspace-scoped packaged resources are dropped without replacement (brief non-goals).
- No competitor references drive this design — it is internal-truth driven (the bundle implementation being absorbed is the reference); `.resources/` citations are therefore absent by design.

## Architecture Decision Records

- [ADR-001: Hard-delete the public Bundle product; Extension is the single kit unit](adrs/adr-001.md)
- [ADR-002: Package automation with enable-as-consent](adrs/adr-002.md)
- [ADR-003: MCP-shaped extension env binding](adrs/adr-003.md)
- [ADR-004: Extension-owned resource records; kit scope is global-only](adrs/adr-004.md)
- [ADR-005: Network confirm on the extension lifecycle; update refuses before apply](adrs/adr-005.md)
- [ADR-006: Inventory and preview as dedicated read-only verbs and routes](adrs/adr-006.md)
- [ADR-007: Dir-per-agent static agent contract with sidecars](adrs/adr-007.md)
- [ADR-008: Code-first static resource declaration](adrs/adr-008.md)

## References

- `_brief.md` — requirements + removal inventory (paths corrected above where drifted).
- Baseline program: `.compozy/tasks/ext-improvs/_techspec.md` + `adrs/adr-001..008.md`.
- Absorbed implementations (implementers read these before deleting them): `internal/extension/bundle.go`, `bundle_agent_loader.go`, `bundle_job.go`, `bundle_layout_loader.go`, `bundle_decode.go`, `internal/bundles/service_activation.go`, `service_resolution.go`, `service_materialize_{agents,automation,layout,bridge}.go`, `service_validate_agents.go`, `network_requirement.go`, `network_requirement_reconcile.go`, `resource.go`, `resource_store*.go`.
- Patterns mirrored: `internal/settings/mcp_secrets.go` + `mcp_catalog_install.go` + `internal/vault/mcp_refs.go` (binding), `internal/daemon/loop_resources.go` (declaration provider), `internal/extension/lifecycle_events.go` (event matrix).
- Current-shape inventory: `internal/extension/{manifest.go,manifest_normalize.go,manifest_load.go,manifest_encode.go,build_describe.go,manager_supervision.go,manager_resource_loading.go,manager_env_resolution.go,manager.go,describe.go,surfaces/registry.go,registry.go,registry_lifecycle.go}`, `internal/daemon/{extensions.go,agent_skill_publisher.go,agent_skill_catalog.go,agent_skill_desired_resources.go,boot_automation_bundles.go,extension_doctor.go via api/core}`, `internal/automation/{model/types.go,manager_definition_sync.go,manager_runtime_apply.go}`, `internal/windowmanager/{layout.go,resource_codec.go,service_layout_resource.go}`, `internal/session/soul_start.go`, `internal/soul`, `internal/heartbeat`, `internal/vault/{types.go,mcp_refs.go,service.go}`, `internal/store/globaldb/schema/definitions/{33_extensions.sql,34_resources.sql,30_automation.sql,39_vault.sql}`.
- Memory: `docs/_memory/standing_directives.md` (SD-002/006/008/011), `docs/_memory/lessons/{L-001,L-003,L-004,L-005,L-006,L-007,L-008,L-012}.md`, `docs/_memory/glossary.md`.
