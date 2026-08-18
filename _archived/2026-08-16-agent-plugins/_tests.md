# Test Specification: Agent Plugins Ingestion

Canonical test contract for Agent Plugins ingestion. Companion to `_spec.md`. Derived from `_user_stories.md` (behavior), `_spec.md` Part II (components), `_dx.md` (CLI/API journeys), and `_uiux.md` (browser journeys).

## Strategy

- Frameworks/harnesses: standard Go testing, table-driven, `t.Run("Should …")` + `t.Parallel` (no `t.Setenv` in parallel tests), `-race`; fakes only at I/O boundaries (vault resolver, `httptest` MCP endpoint, staged-error injection for interruption cases). Web: Vitest for components, Playwright for browser E2E on the daemon-served SPA. Runtime E2E on the existing `acpmock` harness.
- Placement: canonical suites only — `internal/extension/agentplugin/` (new, co-located), `internal/extension/` (manifest/synthesis/registry integration), `internal/registry/` (installer detection), `internal/config/` (server widening), `internal/cli/` (output), `internal/daemon/` (integration + distribution E2E), `internal/api/httpapi/` (transport parity — `transport_parity_integration_test.go`, owner of IT-012), `internal/marketplace/` (feed), `web/src/systems/{extensions,marketplace}/` (Vitest), `web/e2e/` (Playwright). No standalone regression files where a canonical suite owns the invariant.
- Fixtures: one committed conformant testdata package (2 valid skills, 1 invalid skill, stdio + streamable-http + sse + malformed server entries), adversarial variants imported from the reference clients: plugin root path containing a literal `${PLUGIN_DATA}` (self-reference), 8-bad-1-good server isolation set, real symlink escapes, real `git init` + `file://` install. **No live network in any lane.**
- Existing-gate ownership: archive hardening (size caps, traversal, content scan, digest) and network-failure fetch paths are owned by the existing `internal/registry` suites — rows below annotate rather than duplicate.

## Coverage Matrix

| Source | Behavior | Unit | Integration | E2E |
| --- | --- | --- | --- | --- |
| US-001 | Local-dir install, verbatim identity, inert kit | UT-042 | IT-001 | E2E-002 |
| US-001.AC-3 | Dotted name verbatim everywhere | UT-005 | IT-001 | — |
| US-001.AC-4 | Empty package installs with empty resource set | UT-025 | IT-001 | — |
| US-001.EC-1 | Unsupported vs unrelated `$schema` errors | UT-002, UT-003 | IT-003 | — |
| US-001.EC-2 | Name-constraint violations fatal | UT-006 | — | — |
| US-001.EC-3 | Wrong-typed field fatal; unknown field ignored+reported; null ≠ absent | UT-007, UT-008, UT-009 | — | — |
| US-001.EC-4 | Name collision → existing-instance error, no partial state | — | IT-014 | — |
| US-001.EC-5 | Symlink escapes refused per ladder | UT-004, UT-020 | — | — |
| US-001.EC-6 | Archive caps/traversal | — (owned: `internal/registry` hardening suites) | — | — |
| US-001.EC-7 | Interrupted install leaves no partial state | — | IT-016 | — |
| US-001.EC-8 | Concurrent same-name installs → one instance | — | IT-014 | — |
| US-001.EC-9 | AWS-kit-scale package (30+ skills), no report truncation | — | IT-001 (30-skill fixture variant) | — |
| US-002 | Git/GitHub/archive sources auto-detect | — | IT-001 (git variant) | E2E-001 |
| US-002.EC-1 | Single-wrapper descent | UT-038 | — | — |
| US-002.EC-2 | No recognized layout → three-roots error | UT-039 | — | — |
| US-002.EC-3 | Network failure mid-fetch | — (owned: `internal/registry` fetch suites) | — | — |
| US-002.EC-4 | Same name from second source → conflict | — | IT-014 | — |
| US-003 | Client-specific layout deterministic failure | — | IT-003 | E2E-003 |
| US-003.EC-1 | Conformant root beats client-specific dir | UT-041 | — | — |
| US-003.EC-2 | Symlink/non-regular plugin.json = absent | UT-004 | — | — |
| US-003.EC-3 | Same failure machine-readable for agents | — | IT-013 | — |
| US-004 | Dual manifest → native wins + note | UT-029, UT-045 | — | E2E-003 |
| US-004.EC-1 | Invalid native + valid portable → native error, no fallback | UT-030 | — | — |
| US-004.EC-2 | Lifecycle ops on dual-manifest = pure native | — | IT-004 | — |
| US-005 | Component failure isolation with visible reasons | UT-012, UT-013, UT-018 | IT-001 | E2E-002 |
| US-005.AC-3 | sse skipped vs malformed sse — two messages | UT-017 | — | — |
| US-005.AC-4 | `extensions` namespaces ignored without validation | UT-009, UT-010 | — | — |
| US-005.AC-5 | Skips visible on install output + status | UT-042 | IT-012 | — |
| US-005.EC-1 | `skills` location is a file → component disabled | UT-014 | — | — |
| US-005.EC-2/EC-3 | mcp.json unreadable / unsupported schema → component disabled | UT-016 | — | — |
| US-005.EC-4 | Nested skill never discovered | UT-011 | — | — |
| US-005.EC-5 | Fully-degraded install succeeds, reasons visible | UT-025 | IT-001 (degraded variant) | — |
| US-005.EC-6/EC-7 | Duplicate/adversarial server keys deterministic | UT-026 | — | — |
| US-006 | Skills work in sessions across providers | — | IT-004, IT-005 | E2E-001 |
| US-006.AC-2 | Load-time content scan applies | — (owned: `internal/skills` VerifyContent suites) | — | — |
| US-006.EC-1 | Skill name shadowing follows precedence | — | IT-004 (owned: skill-precedence suites, annotated) | — |
| US-006.EC-2 | Sidecar containment | — (owned: `internal/skills` symlink suites) | — | — |
| US-006.EC-3 | Running session unaffected by enable | — | IT-004 | — |
| US-007 | MCP servers work with env contract | UT-015, UT-022 | IT-005, IT-006 | E2E-001 |
| US-007.AC-2 | URL policy enforced | UT-024 | — | — |
| US-007.AC-3 | Data persists across restart/update | — | IT-007 | — |
| US-007.AC-4 | Reserved env + single-pass expansion | UT-021, UT-022 | — | — |
| US-007.EC-1 | Host env never interpolated | — | IT-005 | — |
| US-007.EC-2 | PATH-miss launch failure per-server, visible | — | IT-006 | — |
| US-007.EC-3 | cwd containment escape skipped at ingest | UT-020 | — | — |
| US-007.EC-4 | Data-dir creation failure degrades stdio only | — | IT-017 | — |
| US-007.EC-5 | Server-name collision with session config deterministic | — | IT-005 | — |
| US-008 | Enable/disable publication | — | IT-004 | E2E-002 |
| US-008.EC-1 | Degraded enable shows zero-resources + reasons | UT-049 | IT-004 | — |
| US-008.EC-2 | Disable leaves running sessions | — | IT-004 | — |
| US-008.EC-3 | Enable/disable racing update serializes | — | IT-014 | — |
| US-009 | Update refreshes + preserves data | — | IT-007 | E2E-002 |
| US-009.EC-1 | Drifted layout update fails, install intact | — | IT-008 | — |
| US-009.EC-2 | Update ladder re-runs, diagnostics replaced | — | IT-007 | — |
| US-009.EC-3 | Update while disabled preserves state | — | IT-007 | — |
| US-009.EC-4 | No-op update | — | IT-007 | — |
| US-010 | Remove deletes instance + data | UT-044 | IT-009 | E2E-002 |
| US-010.EC-1..EC-4 | Remove while enabled / absent data / deletion failure / re-run converges | — | IT-009 | — |
| US-011 | Secrets bind for remote servers | UT-046 | IT-011, IT-006 | — |
| US-011.EC-1 | Dangling vault ref → existing binding error | — | IT-011 | — |
| US-011.EC-2 | Remove drops bindings | — | IT-009 | — |
| US-011.EC-3 | Credential-shaped package values refused | UT-023, UT-036 | — | — |
| US-012 | Validate full ladder, stateless, exit codes | UT-043 | IT-013 | E2E-002, E2E-003 |
| US-012.EC-1 | Validate native dir unchanged | UT-043 | — | — |
| US-012.EC-2 | Validate dual-manifest as native + note | UT-029, UT-045 | — | — |
| US-012.EC-3 | Validate client layout = install error | — | — | E2E-003 |
| US-013 | Dev link + reload iteration | — | IT-010 | — |
| US-013.EC-1 | Fatal edit keeps last-good | — | IT-010 | — |
| US-013.EC-2 | Workspace/global precedence visible | — | IT-010 | — |
| US-014 | Agent end-to-end with structured output | — | IT-012, IT-013 | — |
| US-014.EC-1 | Non-interactive consent fails deterministically | UT-042 | IT-013 | — |
| US-014.EC-2 | Not-found shape identical across transports | — | IT-012 | — |
| US-014.EC-3 | Concurrent agent ops serialize | — | IT-014 | — |
| US-015 | Format + diagnostics on all surfaces | UT-048, UT-049 | IT-012 | E2E-004 |
| US-015.EC-1 | Degraded empty state explicit | UT-049 | — | — |
| US-015.EC-2 | Badge absent on native | UT-048 | — | — |
| US-015.EC-3 | Web/CLI parity | — | IT-012 | — |
| US-016 | Catalog entry with badge installs via web | UT-050 | IT-015 | E2E-004 |
| US-016.EC-1 | Drifted entry fails legibly in dialog | — | — | E2E-005 |
| US-016.EC-2 | Marker-less portable entry still installs | — | IT-015 | — |
| US-016.EC-3 | Resource-only consent shows no permission grants | — | IT-015 (owned: trust-dialog suites, annotated) | — |
| Component: `agentplugin` reader | classify/name/ladder/skills/mcp/expansion/version-match/cwd domains | UT-001–UT-026, UT-055, UT-056 | — | — |
| Component: synthesis + load branch | mapping/precedence/waiver/determinism/diagnostics + total ordering | UT-027–UT-033, UT-057 | IT-001 | — |
| Component: config widening + `mcppolicy` | headers fields/source-aware policy/sidecar/secret refs | UT-034–UT-037, UT-052 | IT-006 | — |
| Component: registry detection | third root/triage/wrapper/dual-root matrix | UT-038–UT-041 | — | — |
| Component: CLI output | blocks/validate/remove/note/bind/modes/summary | UT-042–UT-047, UT-054 | — | — |
| Component: data-dir encoding (B-001) | dedicated root, `data`-named package, instance keys | UT-053 | IT-001 | — |
| Component: web components | badge/skipped rows/card | UT-048–UT-050 | — | E2E-004 |
| Component: marketplace feed | format field, hard cut at current version | UT-051 | IT-015 | — |
| Component: persistence migration | new columns fresh/reopen | — | IT-018 | — |

## Unit Tests

### `internal/extension/agentplugin` (Spec: Core Interfaces — conformance reader)

- **UT-001** (happy): `ClassifyManifest` — dir with `$schema` exactly `https://agent-plugins.org/schemas/1.0.0/plugin.schema.json` → `SchemaSupported`.
- **UT-002** (error): `ClassifyManifest` — `$schema` `https://agent-plugins.org/schemas/2.0.0/plugin.schema.json` → `SchemaUnsupportedVersion`, declared string returned verbatim.
- **UT-003** (error): `ClassifyManifest` — `$schema` `https://example.com/other.json` or missing → `SchemaUnrelated`.
- **UT-004** (error): root `plugin.json` as symlink and as directory → treated as absent (no manifest), error path per detection.
- **UT-005** (happy, table): `ValidateName` accepts `a`, `acme.tools`, `my-plugin`, `lint3r`, 64-char name.
- **UT-006** (error, table): `ValidateName` rejects empty, `My-Plugin`, `-start`, `end.`, `has--double`, `too..dots`, 65 chars, name with non-ASCII byte (`café`).
- **UT-007** (error): `Load` fatal — missing `name`; numeric `homepage`; `keywords` with non-string element → `*ManifestError` with one issue each naming the field.
- **UT-008** (error): `Load` — explicit `"version": null` → fatal; absent `version` → loads fine (null ≠ absent).
- **UT-009** (happy): `Load` — unknown top-level field `"icon"` → popped, diagnostic `manifest: ignored unknown top-level field: icon`, load succeeds.
- **UT-010** (happy): `Load` — `"extensions": 42` → dropped + diagnostic; `extensions.com.example.x` object with garbage contents → ignored, zero validation, zero diagnostics.
- **UT-011** (happy): skills discovery — `skills/review/SKILL.md` + `skills/group/nested/SKILL.md` → only `review` in `Package.Skills`.
- **UT-012** (error): skill with frontmatter `name: other-name` in dir `deploy` → skipped, diagnostic `skill:deploy: name must match the directory …`, sibling `review` survives.
- **UT-013** (error): skill with unterminated frontmatter / missing `description` → skipped with distinct messages, scope `skill:<dir>`.
- **UT-014** (boundary): `skills` is a regular file → diagnostic `skills: skills must be an in-root directory`, MCP ingestion continues.
- **UT-015** (happy): mcp.json golden translation — stdio (`command:"./bin/validator"`, args with `${PLUGIN_DATA}`, env with `${PLUGIN_ROOT}`, cwd `${PLUGIN_ROOT}`) and streamable-http (`url` + `X-Tenant` header) → `ServerSpec`s with expanded absolute paths, `PLUGIN_ROOT`/`PLUGIN_DATA` injected into env last.
- **UT-016** (error): mcp.json with unsupported `$schema` / non-object root / missing `mcpServers` → MCP component disabled (`mcp:` scope diagnostic), skills unaffected — asymmetric with the fatal plugin.json rule.
- **UT-017** (error): well-formed `sse` entry → `mcp:legacy-events: sse transport is not supported`; malformed `sse` (missing url) → `mcp:legacy-events: invalid mcp server entry` — two distinct messages.
- **UT-018** (error): 8-bad-1-good isolation set (unknown type, missing command, bad cwd, reserved env, bad header, bad URL, whitespace command, extra field) → `Package.Servers` keys exactly `["valid"]`, 8 scoped diagnostics.
- **UT-019** (error, table): command token — `"my tool"` (whitespace), `bin/tool` (slash, not `./`), `C:tool.exe`, `""` → per-entry skip with the command-token message; bare `validator` and `./bin/validator` accepted.
- **UT-020** (error): cwd `./../escape`, cwd resolving through a real symlink outside root, mixed base `./${PLUGIN_DATA}/x` → skipped with containment messages.
- **UT-021** (error): entry env declaring `PLUGIN_ROOT` (and `PLUGIN_DATA`) → that server skipped with the reserved-variable message.
- **UT-022** (happy/boundary): expansion — plugin root path itself containing literal `${PLUGIN_DATA}` → single-pass result contains no double expansion; `${UNKNOWN}` stays literal; `command` containing `${PLUGIN_ROOT}` is not expanded (and is rejected as a path form).
- **UT-023** (error, table): headers — duplicate `X-Tenant`/`x-tenant`, non-tchar name `"Bad Header"`, value with `\r\n` → invalid entry; credential-shaped `authorization: Bearer …` declared by the package → refused with scoped diagnostic.
- **UT-024** (error, table): URL policy — `http://api.example.com` (non-loopback http), `https://user:pw@x.com`, `https://x.com/mcp#frag` → skip; `http://localhost:3000/mcp` and `http://127.0.0.1/mcp` accepted.
- **UT-025** (boundary): manifest-only package → `Load` succeeds, zero skills, zero servers, zero diagnostics.
- **UT-026** (state): server key literally `mcpServers` treated as an ordinary name; raw JSON duplicate keys resolve deterministically (documented last-wins) — same input, same output across two loads.

### `internal/extension` — synthesis + load branch (Spec: Implementation Design)

- **UT-027** (happy): `SynthesizeAgentPluginManifest` golden — identity fields mapped, `Resources.Skills` lists explicit per-skill `SKILL.md` paths (never the `skills/` dir), stdio → `MCPServerConfig{Command,Args,Env}`, streamable-http → `{Transport:"http",URL,Headers}`, empty provides/permissions/subprocess.
- **UT-028** (happy): `LoadManifest` on a plugin-only dir → adapted manifest, `Format = agent-plugin`.
- **UT-029** (happy): precedence — `extension.toml`+`plugin.json` loads native; `SKILL.md`+`plugin.json` resolves per the documented precedence, deterministically.
- **UT-030** (error): invalid `extension.toml` + valid `plugin.json` → fails with the native manifest error; no fallback to portable.
- **UT-031** (happy): adapted manifest passes validation without `min_compozy_version`; native manifest without it still fails (gate unchanged).
- **UT-032** (state): two `LoadManifest` calls on identical bytes → deep-equal manifests (deterministic synthesis).
- **UT-033** (happy): diagnostics conversion → `contract.DiagnosticItem` with `code=extension_agent_plugin_component_skipped`, `severity=warn`, `evidence.scope` preserved.

### `internal/config` — server widening (Spec: ADR-006)

- **UT-034** (happy): a synthesized/binding-projected `MCPServer` carries `Headers`/`SecretHeaders` through validation and `cloneMCPServer` (deep copy, no aliasing); values in `SecretHeaders` are references, asserted by shape.
- **UT-035** (error): validation matrix — `headers` or `secret_headers` on a stdio server → the exact per-transport error; existing stdio/http arms unchanged (regression rows); the workspace sidecar surface stays closed (see UT-037).
- **UT-036** (error, table): `mcppolicy.ValidateHeaders` source-aware matrix — `content-type`/`mcp-*` rejected for every source; sensitive set (`authorization`, `proxy-authorization`, `cookie`, `set-cookie`) with `SourcePackageFixed` → entry-reject with the stable scoped code; `authorization` with `SourceOperatorSecret` → allowed iff `authEnabled=false`; other sensitive names rejected for operator bindings too; case-insensitive duplicates rejected across the **union** of fixed and secret maps; identical outcomes when called from sidecar validation, ingestion, and the executor (one authority).
- **UT-037** (error): the workspace `mcp.json` sidecar surface is NOT widened (B-005 cut) — `ParseMCPServersJSON` rejects documents declaring `headers` or `secret_headers` with a deterministic error naming the unsupported field, even though the underlying struct carries them (internal wire fields never become sidecar surface by accident).
- **UT-052** (error): secret-reference discipline — `SecretHeaders` values are references before and after clone/JSON/YAML/TOML serialization; a resolved credential value never appears in any serialized form of `MCPServer`, in `secrets list` output, or in extension payload projections (negative assertions on the serialized bytes).
- **UT-053** (boundary, table): data-dir encoding — global `acme.tools` → `extension-data/acme.tools`; dev link → `extension-data/acme.tools@ws-<id>`; a package legally named `data` maps under `extension-data/data` and is **disjoint from every other instance's data root and from the managed install tree**; all encodings are single containment-checked segments.
- **UT-054** (happy): computed `Summary` acknowledges degradation — zero skips → `running and healthy`; one recorded skip → `running (1 component skipped)`; N skips → `running (N components skipped)`.
- **UT-055** (error): cross-document schema-version rule — `plugin.json` at 1.0.0 with an `mcp.json` declaring a different schema version → MCP component disabled with a scoped reason; skills unaffected (standard's version-match rule).
- **UT-056** (happy/boundary, table): `cwd` normalization — omitted `cwd` → the package root; `"${PLUGIN_DATA}"` and `"${PLUGIN_DATA}/state"` are **valid** and resolve inside the data root (second containment domain); `"./sub"` resolves inside the package root; symlink escapes from either domain rejected; mixed-root forms rejected (extends UT-020).
- **UT-057** (state): diagnostic total order — fabricated sets with colliding scopes/servers order deterministically by source set (ingest → live) → scope/server → code → message, identically across two loads and across payload/inventory projections.

### `internal/registry` — detection (Spec: System Architecture)

- **UT-038** (happy): archive/dir with root `plugin.json` (supported schema) accepted as manifest root; single-wrapper directory descent finds it.
- **UT-039** (error): root `plugin.json` with unrelated `$schema`, no other manifest → error listing `extension.toml`, `SKILL.md`, and the Agent Plugins `plugin.json` root.
- **UT-040** (state): `extension.toml` + `SKILL.md` both at root → existing hard error, byte-identical message (regression) — including when `plugin.json` is also present (precedence matrix rule a: the native-vs-native conflict wins, no portable fallback).
- **UT-041** (happy): `plugin.json` + `.claude-plugin/plugin.json` both present → conformant root wins, client dir inert.

### `internal/cli` — output (Spec: `_dx.md` CLI)

- **UT-042** (happy): install success render — `Format:`/`Ingested`/`Skipped` blocks exactly as `_dx.md`; native install output byte-identical to today's golden; structured-output consent rule (`--allow-unverified` without `--yes` → `cli: --allow-unverified requires --yes for structured output`) still holds for plugin sources.
- **UT-043** (happy/error): validate render + exit — warn-only package prints `Status: valid`, `WARN` rows, exits 0; fatal prints `Status: invalid`, `ERROR` rows, exits 1; native directory validation output unchanged; runs daemon-less.
- **UT-044** (happy/error): remove render — two `removed:` lines; data-deletion failure prints the warning with the leftover path and still exits 0.
- **UT-045** (happy): dual-manifest install prints the single `note:` line.
- **UT-046** (error, table): `secrets bind --remote-header` — `deployment-api:Authorization` parses; `noheader`, `:X`, `srv:`, `srv:Bad Header` rejected with flag-usage errors.
- **UT-047** (happy): install and validate payloads render in all four output modes (`human`, `json`, `jsonl`, `toon`) without the missing-formatter runtime error.

### Web components (Vitest, `web/src/systems/…`)

- **UT-048** (happy): `ExtensionFormatBadge` renders `agent plugin` pill for `format="agent-plugin"`, renders nothing for `"compozy"`.
- **UT-049** (happy/boundary): skipped-components section — rows show scope + reason from `diagnostics`; section absent with zero diagnostics; fully-degraded payload renders the explicit zero-resources line.
- **UT-050** (happy): marketplace card renders the format pill from the feed entry's `format`; absent without it.

### `internal/marketplace` — feed (Spec: Data Models)

- **UT-051** (happy): entry with `"format":"agent-plugin"` decodes at the current `manifest_version` (hard cut — reader, feeds, and fixtures updated together); entries without the field default to native; no pre-cut compatibility behavior exists to test.

## Integration Tests

### Install / lifecycle (`internal/extension`, `internal/daemon`)

- **IT-001**: install the conformant testdata package from a local dir — registry row has `format='agent-plugin'`, persisted diagnostics match the fixture's known skips, files moved under `extensions/<name>`, **no data directory exists yet** (created only at first stdio launch — frozen DX contract), kit inert until enable; variants: 30-skill fixture (full component report, no truncation), fully-degraded fixture (row exists, zero resources, all reasons persisted), git-source fixture (real `git init` + `file://`), and a package legally named `data` — installs, updates, and removes without ever touching another instance's data root (B-001 regression).
- **IT-002**: install with fatal manifest (bad name) — no registry row, no install dir, no data dir.
- **IT-003**: install `.claude-plugin` fixture and unrelated-`plugin.json` fixture through the HTTP handler — 422 with `extension_agent_plugin_client_layout` / `extension_agent_plugin_not_manifest`, registry untouched.
- **IT-004**: enable publishes `skill` + `mcp_server` resources owned by the instance; disable removes only those; running-session snapshot unaffected; dual-manifest install behaves as pure native throughout; degraded instance enables with zero resources.
- **IT-005**: session start (degraded/acp path) — merged MCP list carries the package's stdio server with absolute `PLUGIN_ROOT`/`PLUGIN_DATA` in env and literal `${UNKNOWN}` untouched; a session-config server with the same name resolves by the documented precedence, visibly logged.
- **IT-006**: executor client connects to an `httptest` streamable-http server — received request carries the package's `X-Tenant` header plus the bound secret header resolved from a fake vault **at connection time, inside the executor** (the declarative config still holds only the reference afterwards); secret value absent from status/API/`secrets list` reads (redaction); PATH-miss stdio launch failure and remote auth/connection failure each surface as a **live** `extension_mcp_server_unhealthy` diagnostic on status/inventory for exactly that server, listed after the ingest set, without touching siblings. Runtime-health registry lifecycle (B-004): a subsequent successful exchange **clears** the entry; an injected out-of-order old failure cannot overwrite a newer success (monotonic per-key ordering); reload/update/disable/remove evict the instance+generation's entries; a daemon restart starts with an empty registry.
- **IT-007**: update from a changed git fixture — ladder re-runs, persisted diagnostics replaced, data-dir bytes identical before/after, disabled state preserved; unchanged upstream → no-op.
- **IT-008**: update from a fixture restructured to `.claude-plugin` layout — fails with the layout code; prior install still loads and serves resources.
- **IT-009**: remove — registry row, install dir, data dir, and secret bindings gone; re-install starts clean; deletion failure exercised by **injecting a failing filesystem remover at the I/O boundary** (permission tricks are unreliable across macOS/CI) — the residue is quarantined under a non-reusable name, removal completes reporting the quarantine path, and a subsequent same-name install verifiably starts with an **empty** data directory; with both remover and quarantine-rename injected to fail, the removal itself fails deterministically and the instance remains; one real containment integration case stays; repeated remove converges.
- **IT-010**: dev-link the package dir into a workspace — the `extension_dev_links` row carries `format='agent-plugin'` + its own diagnostics; edit a skill + reload → the dev link's diagnostics are replaced atomically with the new `bundle_generation`; fatal manifest edit → reload fails, last-good generation and its diagnostics stay active; **isolation matrix**: a global-native extension and a same-named workspace plugin dev link project distinct `format`/`diagnostics` per `InstanceKey` across CLI, HTTP, UDS, and web reads, and two workspaces with different dev links of one name never leak state.
- **IT-011**: `secrets bind --remote-header` persists `mcp_server`/`header_name` columns and projects the binding as a **reference** into exactly that server's `SecretHeaders`; the same binding created via `PUT /api/extensions/:name/secrets` (widened contract) and via UDS produces the identical row and identical presence-only read — one domain operation across all three paths; dangling vault ref → existing binding error; `secrets list` shows key name + mapping, never values.
- **IT-018**: migration suite extension — fresh apply and reopen-with-data carry all **six** new columns (`extensions.format`/`.ingest_diagnostics_json`, `extension_dev_links.format`/`.ingest_diagnostics_json`, `extension_env_bindings.mcp_server`/`.header_name`) with correct defaults and existing-row backfills; the both-or-neither `CHECK` rejects partially-mapped binding rows at the storage boundary while both valid shapes insert; ahead/integrity/append-only-identity/declarative-equivalence cases extend the canonical stream suites; `make codegen-check` clean.

### Surfaces / parity (`internal/api`, `internal/daemon`, `internal/cli`)

- **IT-012**: transport parity — same instance read via CLI `-o json`, HTTP, and UDS returns identical `format` + `diagnostics` on the extension payload, on **list rows** (one list contract), and on the widened inventory payload; unknown name returns the identical not-found shape on all three. Owning suite: the existing `internal/api/httpapi` transport-parity suite (`transport_parity_integration_test.go`), extended — not duplicated.
- **IT-013**: native tools — `compozy__extensions_install` returns the payload with `format`/`diagnostics`; failure returns the branchable `extension_agent_plugin_*` code; `compozy__extensions_validate` returns `status`/`format`/`would_ingest`/`issues` without daemon state changes; non-interactive unverified install fails with the consent-required error, no hang.
- **IT-014** (concurrency): every race goes through the per-`InstanceKey` lifecycle coordinator — two concurrent installs of one name (one via CLI path, one via HTTP path) → exactly one registry row, loser gets the conflict error; enable/disable racing update serializes to a requested end state, never a hybrid; a dev reload racing a global update on the same name resolves per instance without cross-talk.
- **IT-015**: marketplace flow — a current-version feed fixture with a plugin-format entry projects `format` through `/api/marketplace/*` (hard cut: no bumped/pre-cut fixture exists); installing the entry lands the same payload as a CLI install of the same source; a marker-less portable entry still installs correctly (detection authoritative); resource-only consent surface shows no permission grants (trust-dialog suite annotation).
- **IT-016** (interruption): staged-error injection at each boundary of the coordinator's commit order (after staging; after final move, before registry commit) — no partial instance visible through any read surface; boot reconciliation removes the orphaned final directory (registry is the authority); re-running install succeeds idempotently.
- **IT-017**: data-dir creation failure at **first stdio launch** via an injected failing creator at the I/O boundary — exactly that server reports live `extension_mcp_server_unhealthy` with the creation reason; install/update state, skills, remote servers, and sibling stdio servers unaffected.

## End-to-End Tests

### Runtime distribution (US-002, US-006, US-007) — `internal/daemon` E2E harness

- **E2E-001**: install the fixture package from a local git source → enable → `acpmock` session activates an ingested skill and calls the stdio server, which echoes its env back — asserting absolute `PLUGIN_ROOT`/`PLUGIN_DATA`, expanded args, and data-dir creation-at-first-launch + writability; the journey's assertions walk all 8 minimum-conformance checklist items. **Evidence chain (L-007/L-026):** this deterministic harness proves the daemon-side contract; `qa-execution` owns the real-provider matrix. Claude Code and Hermes must pass the complete skill, stdio, and remote-MCP walk. OpenClaw's verified ACP rejection is recorded as an explicit limitation, not counted as provider delivery. The user-owned Phase 2 listing may cite the conformance and passing transport evidence after the interoperability docs are deployed.

### CLI journeys (US-001, US-005, US-008–US-010, US-012) — `_dx.md` transcripts verbatim

- **E2E-002**: golden path — `install ./fixture` (blocks output) → `enable` → `status` (Format + Diagnostics) → `validate` (exit 0 with WARN) → `update` (no-op) → `remove` (two `removed:` lines); each step's stdout matched against the `_dx.md` transcript shapes.
- **E2E-003**: failure paths — `install ./claude-layout-fixture` (client-layout message), `install ./Bad--Name` (fatal name message), `install ./dual-target` (note line), `validate ./claude-layout-fixture` (same layout error); exit codes 1/1/0/1.

### Browser journeys (US-015, US-016) — Playwright, `web/e2e`

- **E2E-004**: marketplace card shows the `agent plugin` pill → install dialog (unchanged fields) → trust dialog consent for an unverified entry → installed card + settings detail show the badge → inventory panel lists kit items and the Skipped section with the fixture's sse reason.
- **E2E-005**: catalog entry pointing at a drifted (client-layout) fixture → web install fails and the dialog renders the deterministic layout message legibly (no toast-only failure).
