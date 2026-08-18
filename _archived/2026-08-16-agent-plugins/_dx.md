# Developer Experience: Agent Plugins Ingestion

Public-surface contract for ingesting Agent Plugins packages. Companion to `_spec.md` (Part II serves this surface) and `_tests.md` (E2E journeys use these exact invocations). Everything below is written as if shipped. No new verbs, no new routes: the surface is the existing extension surface, extended where the format needs it.

## Golden Path

Install a portable package from a local checkout, enable it, use it in a session — 30 seconds:

```console
$ compozy extension install ./acme.tools
✓ install acme.tools

Format: agent plugin
Ingested
========
Kind:        Name:              Detail:
skill        summarize          skills/summarize
skill        deploy             skills/deploy
mcp_server   local-validator    stdio
mcp_server   deployment-api     streamable-http
Skipped
=======
mcp_server   legacy-events      sse transport is not supported
next: compozy extension enable acme.tools

Extension
=========
Name:     acme.tools
Version:  1.2.0
Type:     resource
Format:   agent plugin
Source:   user
Enabled:  false
State:    disabled
Daemon:   running
Summary:  disabled

$ compozy extension enable acme.tools
✓ enable acme.tools
next: compozy extension status acme.tools

$ compozy session start
# in the session: the `summarize` and `deploy` skills activate like any
# extension skill; `local-validator` and `deployment-api` tools are callable.
```

## Package layout (what plugin authors write)

The package is a plain directory in the Agent Plugins standard layout. Compozy authors nothing new — this is the third-party format, shown complete because `validate` and `install` consume it exactly as written:

```text
acme.tools/
├── plugin.json          # required — the portable manifest
├── skills/              # optional — one skill per immediate child directory
│   ├── summarize/
│   │   └── SKILL.md
│   └── deploy/
│       ├── SKILL.md
│       └── scripts/rollback.sh
├── mcp.json             # optional — MCP servers
└── com.example.other/   # ignored — another client's namespace directory
```

`plugin.json` — only `$schema` and `name` are required:

```json
{
  "$schema": "https://agent-plugins.org/schemas/1.0.0/plugin.schema.json",
  "name": "acme.tools",
  "version": "1.2.0",
  "description": "Acme deployment skills and validators",
  "license": "MIT",
  "repository": "https://github.com/acme/acme-tools"
}
```

`skills/summarize/SKILL.md` — agentskills.io format; `name` must equal the directory name:

```markdown
---
name: summarize
description: Summarize deployment reports into release notes.
---

Read the report the user names and produce a summary…
```

`mcp.json` — transports `stdio` and `streamable-http` are ingested; `sse` is skipped with a diagnostic:

```json
{
  "$schema": "https://agent-plugins.org/schemas/1.0.0/mcp.schema.json",
  "mcpServers": {
    "local-validator": {
      "type": "stdio",
      "command": "./bin/validator",
      "args": ["--data", "${PLUGIN_DATA}/validator"],
      "env": { "CONFIG": "${PLUGIN_ROOT}/config.json" },
      "cwd": "${PLUGIN_ROOT}"
    },
    "deployment-api": {
      "type": "streamable-http",
      "url": "https://deploy.example.com/mcp",
      "headers": { "X-Tenant": "public-tenant" }
    },
    "legacy-events": { "type": "sse", "url": "https://legacy.example.com/sse" }
  }
}
```

Runtime contract the package can rely on: `PLUGIN_ROOT` and `PLUGIN_DATA` are absolute paths; `${PLUGIN_ROOT}`/`${PLUGIN_DATA}` expand (single-pass) in `args`, `env` values, and `cwd` only; the data directory is `~/.compozy/extension-data/acme.tools/` (a dedicated root, never inside the install tree), created when a stdio server first needs it, preserved across updates, deleted on remove.

## CLI

Every verb below already exists; all emit structured output via the global `-o human|json|jsonl|toon`. Exit code 0 on success, 1 on error, everywhere.

### `compozy extension install <source>`

Sources: local path (`./acme.tools`), `github:owner/repo[@ref]`, `git:<url>`, bare `owner/repo` (curated, GitHub fallback), archives. Format detection is automatic per source content — no flag. Flags unchanged (`--version`, `--asset`, `--allow-unverified`, `--yes`).

Human output on success: shown in Golden Path. The `Ingested`/`Skipped` blocks appear only for agent-plugin-format installs; native installs keep today's output byte-for-byte.

`-o json` returns the extension payload with two additions — `format` and populated `diagnostics`:

```json
{
  "name": "acme.tools",
  "workspace_id": "",
  "version": "1.2.0",
  "type": "resource",
  "format": "agent-plugin",
  "source": "user",
  "enabled": false,
  "state": "disabled",
  "capabilities": [],
  "permissions": [],
  "requires_env": [],
  "missing_env": [],
  "bound_env_keys": [],
  "network_requirement_digest": "",
  "network_confirmation_required": false,
  "daemon_running": true,
  "update_available": false,
  "diagnostics": [
    {
      "id": "extension/acme.tools/agent-plugin/mcp:legacy-events",
      "code": "extension_agent_plugin_component_skipped",
      "severity": "warn",
      "category": "extension",
      "title": "Component skipped",
      "message": "mcp server \"legacy-events\": sse transport is not supported",
      "data_freshness": "live",
      "evidence": { "scope": "mcp:legacy-events", "component": "mcp_server" }
    }
  ]
}
```

`format` is always present on extension payloads: `"compozy"` for native extensions, `"agent-plugin"` for ingested packages.

Failure — client-specific layout (the most common real-world miss):

```console
$ compozy extension install ./aws-core
error: extension: ./aws-core is a Claude Code plugin (.claude-plugin/plugin.json), not an Agent Plugins package; Compozy installs the standard layout (plugin.json at the package root) — see https://agent-plugins.org/plugin-authors and the migrate-agent-plugin guide
$ echo $?
1
```

Failure — root `plugin.json` from another ecosystem, no other manifest:

```console
$ compozy extension install ./random-npm-thing
error: extension: ./random-npm-thing has no recognized manifest; accepted roots are extension.toml, SKILL.md, or an Agent Plugins plugin.json ($schema https://agent-plugins.org/schemas/1.0.0/plugin.schema.json)
```

Failure — unsupported standard version:

```console
$ compozy extension install ./future-pkg
error: extension: ./future-pkg declares Agent Plugins schema "https://agent-plugins.org/schemas/2.0.0/plugin.schema.json"; this daemon supports 1.0.0
```

Failure — fatal manifest error (name constraint):

```console
$ compozy extension install ./Bad--Name
error: extension: agent plugin manifest invalid: name "Bad--Name" must be 1-64 lowercase letters, digits, dots, or hyphens, start and end alphanumeric, without "--" or ".."
```

Dual manifest in one directory — native wins, one notice line:

```console
$ compozy extension install ./dual-target
✓ install dual-target
note: directory carries both extension.toml and plugin.json; installed as a Compozy extension (native manifest wins)
next: compozy extension status dual-target
…
```

The full precedence matrix: `extension.toml` + `SKILL.md` both at root stays today's hard error (with or without `plugin.json`); exactly one native root beats `plugin.json`; `plugin.json` is selected only when no native root exists; a selected invalid root fails — never falls through to another format.

Trust behavior is unchanged: unverified sources still require `extensions.trust.allow_unverified` + `--allow-unverified` (+ `--yes` for structured output), same consent prompt, same `extension_checksum_unverified` / `extension_unverified_policy_blocked` errors.

### `compozy extension validate [directory]`

Auto-detects the layout like install. For an agent-plugin directory it runs the full standard conformance ladder without touching daemon state:

```console
$ compozy extension validate ./acme.tools
Extension bundle validation
===========================
Status:   valid
Format:   agent plugin
Package:  acme.tools 1.2.0
Issues:   1
Would ingest:
  skill        summarize
  skill        deploy
  mcp_server   local-validator    stdio
  mcp_server   deployment-api     streamable-http
WARN mcp:legacy-events:  sse transport is not supported
$ echo $?
0
```

Fatal problems set `Status: invalid`, print `ERROR <scope>: <message>` rows, and exit 1 — same contract as native bundle validation (component skips are `WARN` and exit 0; a package that would be rejected exits 1):

```console
$ compozy extension validate ./Bad--Name
Extension bundle validation
===========================
Status:   invalid
Format:   agent plugin
Issues:   1
ERROR plugin.json:  name "Bad--Name" must be 1-64 lowercase letters, digits, dots, or hyphens, start and end alphanumeric, without "--" or ".."
error: cli: extension bundle validation failed
$ echo $?
1
```

`-o json`:

```json
{
  "status": "valid",
  "format": "agent-plugin",
  "name": "acme.tools",
  "version": "1.2.0",
  "would_ingest": [
    { "kind": "skill", "name": "summarize" },
    { "kind": "skill", "name": "deploy" },
    { "kind": "mcp_server", "name": "local-validator", "transport": "stdio" },
    { "kind": "mcp_server", "name": "deployment-api", "transport": "streamable-http" }
  ],
  "issues": [
    { "severity": "warn", "scope": "mcp:legacy-events", "message": "sse transport is not supported" }
  ]
}
```

### `compozy extension status <name>` / `list` / `inventory`

`status` gains the `Format:` line and a `Diagnostics` block when skips were recorded:

```console
$ compozy extension status acme.tools
Extension
=========
Name:         acme.tools
Version:      1.2.0
Type:         resource
Format:       agent plugin
Source:       user
Enabled:      true
State:        active
Daemon:       running
Summary:      running (1 component skipped)
Diagnostics
===========
WARN mcp:legacy-events:  sse transport is not supported
```

`list` keeps today's table columns (an ingested package shows `Type: resource` like any resource-only extension); `-o json` list items are the same full extension payload as `status` — `format` and `diagnostics` included (one contract across CLI list, `GET /api/extensions`, and `compozy__extensions_list`). `inventory` is the kit view and owns the skipped-components presentation — it gains the `Format:` line and a `Skipped` section (structured output carries the same `format` + `diagnostics` fields):

```console
$ compozy extension inventory acme.tools
Extension inventory
===================
Extension:  acme.tools
Format:     agent plugin
Enabled:    true
Kind:        Name:              ID:                                          Live:
skill        summarize          extension/acme.tools/skills/summarize        true
skill        deploy             extension/acme.tools/skills/deploy           true
mcp_server   local-validator    extension/acme.tools/mcp/local-validator     true
mcp_server   deployment-api     extension/acme.tools/mcp/deployment-api      true
Skipped
=======
mcp_server   legacy-events      sse transport is not supported
```

Runtime failures (a stdio command missing from PATH, a remote server refusing auth) surface as **live** diagnostics on `status`/`inventory` with their own code (`extension_mcp_server_unhealthy`), listed after the recorded ingest skips — package validity and runtime availability are never conflated, and one unhealthy server never marks its siblings.

### `compozy extension update` / `remove`

Both unchanged in shape. Update re-runs the full ladder and reports like install; the data directory survives:

```console
$ compozy extension update acme.tools
✓ update acme.tools
Format: agent plugin
Ingested
========
…
note: data directory preserved (~/.compozy/extension-data/acme.tools)
```

Remove deletes the data directory and says so:

```console
$ compozy extension remove acme.tools --global
✓ remove acme.tools
removed: ~/.compozy/extensions/acme.tools
removed: ~/.compozy/extension-data/acme.tools
```

If data-directory deletion fails, the residue is quarantined so the name stays clean for reinstall: `warning: data directory could not be removed; quarantined as ~/.compozy/extension-data/acme.tools.removed-1755280000 — a fresh install of acme.tools starts empty`. If even the quarantine rename fails, the removal itself fails with a deterministic error and the instance stays installed — a completed remove never leaves reachable data under the name.

### `compozy extension secrets bind` (authenticated remote servers)

Bindings work exactly as today for stdio servers (bound env reaches the server process). For a remote server that needs a credential, the binding attaches to a header on one named server:

```console
$ compozy extension secrets bind acme.tools \
    --env DEPLOY_API_TOKEN \
    --vault-ref vault:extensions/acme.tools/deploy_api_token \
    --remote-header deployment-api:Authorization
✓ bind acme.tools DEPLOY_API_TOKEN
```

Sessions connecting to `deployment-api` send `Authorization: <bound value>` — the daemon resolves the vault reference at connection time, on its side of the boundary; the value never lands in any config file or provider process (store the full header value, e.g. `Bearer eyJ…`, in the vault). Public reads keep exposing key names only:

```console
$ compozy extension secrets list acme.tools
Keys:  DEPLOY_API_TOKEN (remote-header deployment-api:Authorization)
```

A package that ships a sensitive header (`authorization`, `proxy-authorization`, `cookie`, `set-cookie`) in `mcp.json` gets **that server entry skipped** with a `WARN` diagnostic naming the entry — refused, never silently stripped; the binding path is the only credential path.

### `compozy extension dev [directory]`

Unchanged verb; a portable package directory dev-links like any extension source. `--watch` re-validates on change; a fatal manifest edit keeps the last good generation active and prints the fatal error.

## HTTP / UDS API

No new routes. Existing routes carry the new fields; HTTP and UDS are identical.

`POST /api/extensions` — request unchanged:

```json
{ "source": "local_path", "ref": "/home/alex/acme.tools" }
```

`201` response — `{"extension": {…}}` with `format` and `diagnostics` as in the CLI JSON above.

`422` — fatal manifest ladder, validation-error envelope:

```json
{
  "error": "extension: agent plugin manifest invalid",
  "diagnostic": { "code": "extension_agent_plugin_manifest_invalid", "severity": "error", "…": "…" },
  "issues": [
    { "severity": "error", "path": "plugin.json", "message": "name \"Bad--Name\" must be 1-64 lowercase letters, digits, dots, or hyphens, start and end alphanumeric, without \"--\" or \"..\"" }
  ]
}
```

`422` — client-specific layout / unrecognized manifest / unsupported schema version use the generic error envelope with their codes (table below), e.g.:

```json
{ "error": "extension: /srv/aws-core is a Claude Code plugin (.claude-plugin/plugin.json), not an Agent Plugins package; …", "code": "extension_agent_plugin_client_layout" }
```

`GET /api/extensions/:name` / `GET /api/extensions` — payloads carry `format` + `diagnostics`. `GET /api/extensions/:name/inventory` — gains the same `format` + `diagnostics` fields beside `items[]` (the kit view owns skipped-component presentation).

`PUT /api/extensions/:name/secrets` — the binding entries gain the optional remote-header target (both fields or neither), the same domain operation the CLI verb calls:

```json
{
  "bindings": [
    {
      "env_name": "DEPLOY_API_TOKEN",
      "vault_ref": "vault:extensions/acme.tools/deploy_api_token",
      "mcp_server": "deployment-api",
      "header_name": "Authorization"
    }
  ]
}
```

`200` echoes the presence-only view (key names + mapping, never values or refs). Agents use this route or the structured CLI verb — deliberately no secrets native tool (secret material stays out of model-visible tool traffic, matching the existing zero-secrets-tool posture).

## config.toml

**No new keys.** The existing trust policy governs plugin-format installs identically:

```toml
[extensions.trust]
allow_unverified = true   # unchanged default; gates unverified sources for every format
```

Format detection is not configurable — a conformant package at any accepted source installs, full stop.

## Native Tools

Same tools, same IDs — payloads gain the same fields:

`compozy__extensions_install` — args `{ "source": { "source": "local_path", "ref": "/home/alex/acme.tools" } }` → result: the extension payload (`format`, `diagnostics` included). Errors are the deterministic codes below — an agent branches on `code`, never prose.

`compozy__extensions_validate` — args `{ "directory": "/home/alex/acme.tools" }` → result: the validate JSON above (`status`, `format`, `would_ingest`, `issues`).

`compozy__extensions_info` / `_list` / `_inventory` — results carry `format` and `diagnostics` per the HTTP payloads.

## Marketplace catalog

A curated feed entry may point at a plugin-format package — one new optional field:

```json
{
  "entry_id": "acme-tools",
  "name": "Acme Tools",
  "description": "Deployment skills and validators for Acme stacks",
  "version": "1.2.0",
  "published_at": "2026-08-20T00:00:00Z",
  "install_slug": "acme/acme-tools",
  "artifact_url": "https://raw.githubusercontent.com/compozy/compozy/main/catalog/artifacts/acme-tools-v1.2.0.tar.gz",
  "digest_sha256": "291157…767",
  "tier": "community",
  "format": "agent-plugin",
  "repository": "https://github.com/acme/acme-tools"
}
```

`format` is display metadata (badge on the card); install-time detection stays authoritative. Absent `format` means native. Web install of such an entry runs the normal flow — install dialog, trust dialog, consent — and lands on the same payload.

## Errors

| Condition | Code | Status | Surface behavior |
| --- | --- | --- | --- |
| Root `plugin.json` is a client-specific package (`.claude-plugin/`, `.codex-plugin/`, `.cursor-plugin/` detected) | `extension_agent_plugin_client_layout` | 422 | Error names the detected layout + points to the standard's migration path; nothing installed |
| No recognized manifest at root | existing `registry: archive missing …` behavior, message now lists all three accepted roots | 422 | Nothing installed |
| `$schema` has the standard's prefix but an unsupported version | `extension_agent_plugin_schema_unsupported` | 422 | Error names declared vs supported version |
| Root `plugin.json` without the standard's `$schema` (unrelated file), no other manifest | `extension_agent_plugin_not_manifest` | 422 | Error says the file is not an Agent Plugins manifest, lists accepted roots |
| Fatal manifest violation (name constraints, wrong-typed field, explicit null, unreadable JSON) | `extension_agent_plugin_manifest_invalid` | 422 | Validation envelope with per-issue rows; nothing installed |
| Component skipped (bad skill, bad server entry, `sse` transport, name/dir mismatch, containment escape, reserved env override) | `extension_agent_plugin_component_skipped` | n/a — not an error | `warn` diagnostic with `evidence.scope`; recorded on the payload, shown by install/status/validate |
| Name already installed | existing conflict behavior | 409 | Existing-instance error; no partial state |
| Unverified source without consent | `extension_checksum_unverified` / `extension_unverified_policy_blocked` (existing) | 422 | Unchanged trust flow |
| Not found (`status`/`remove`/… on unknown name) | existing | 404 | Unchanged |

Two messages are deliberately distinct: a malformed `sse` entry reports `invalid mcp server entry`, a well-formed one reports `sse transport is not supported` — a broken entry never masquerades as merely unsupported.
