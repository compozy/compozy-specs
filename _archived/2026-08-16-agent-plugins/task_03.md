---
status: completed
title: MCP wire + secrets: headers, executor, runtime health, remote-header binding
type: backend
complexity: high
---

# Task 3: MCP wire + secrets: headers, executor, runtime health, remote-header binding

## Overview

Delivers the MCP wire vertical the ingested packages need: the canonical `MCPServer` gains `Headers`/`SecretHeaders` as internal, declarative wire fields (ADR-006), the daemon executor attaches validated fixed headers and resolves secret references at connection time into a non-serializable request header, the runtime-health registry gives live per-server diagnostics their owner (generation-keyed, clear-on-success, evicting), and the `secrets bind --remote-header` vertical lands across CLI, HTTP/UDS, and persistence as one domain operation. Closes with the runtime distribution E2E.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST add `Headers`/`SecretHeaders` to `config.MCPServer` as **declarative internal wire fields**: `SecretHeaders` values are references (vault ref / binding identity), never resolved values; validation arms allow both only on non-stdio transports and delegate to `mcppolicy`; clone deep-copies; every read surface redacts (key names only).
2. MUST keep the workspace `mcp.json` sidecar surface CLOSED (round-2 B-005 cut): the sidecar parser rejects `headers`/`secret_headers` with a deterministic error; `compozy config` MCP verbs and the web MCP editor are untouched.
3. MUST resolve secret references inside the MCP executor immediately before request construction, injecting values only into a local `http.Header` through the existing `authorizationRoundTripper` seam, reusing the existing vault resolution + redaction machinery (`resolveSecretRef`/`registerMCPRequestSecret` pattern); `authorization` interplay with OAuth follows `mcppolicy` (`SourceOperatorSecret` allowed iff no OAuth).
4. MUST implement the runtime-health registry per Part II Monitoring: in-memory, keyed `InstanceKey + bundle generation + server name`, call-site-only writes, monotonic per-key ordering (old failure never overwrites newer success), clear-on-success, eviction on reload/update/disable/remove, restart-empty; projected as `extension_mcp_server_unhealthy` (category `extension`, `data_freshness: "live"`) merged after the ingest set.
5. MUST create the package data directory (path from task_02's accessor) immediately before the first stdio server launch, verify writability, and report creation failure as live health for exactly that server — never ingestion degradation, never an install/update failure.
6. MUST deliver the remote-header binding as ONE domain operation shared by `compozy extension secrets bind --remote-header <server>:<header>` (flag parse + validation), `PUT /api/extensions/:name/secrets` (widened typed binding entries, both-or-neither), and UDS parity — identical rows, identical presence-only reads, identical deterministic errors; OpenAPI + generated TS refresh via `make codegen`.
7. MUST NOT add any `compozy__extensions_secrets_*` native tool (deliberate posture — secret material stays out of model-visible tool traffic; documented in Agent Manageability Plan).
</requirements>

## Subtasks

- [x] 3.1 `config.MCPServer` widening + validation arms + clone + redaction sweep of every read surface
- [x] 3.2 Sidecar closed-surface guard (parse rejection) — `mcpjson.go` stays otherwise untouched
- [x] 3.3 Executor: fixed-header attachment + connection-time secret resolution through the RoundTripper seam
- [x] 3.4 Runtime-health registry (ordering, clear, evict, restart-empty) + projection merge after ingest set
- [x] 3.5 First-launch data-dir creation/verification + live-health failure path
- [x] 3.6 Secrets binding domain op + CLI flag + HTTP/UDS contract widening + OpenAPI/codegen
- [x] 3.7 Session-assembly projection: bindings → `SecretHeaders` references on exactly the target server
- [x] 3.8 Full assigned suites incl. serialization negatives, redaction, ordering races, eviction lifecycle, and the distribution E2E

## Implementation Details

Follow `_spec.md` Part II → Core Interfaces (`MCPServer`, `mcppolicy`), Monitoring (health registry semantics), API Endpoints (secrets contract), Safety Invariants 5–7. Existing anchors: `internal/config/mcp_server.go` + `mcp_server_validation.go:24-42` (per-transport arms), `internal/config/mcpjson.go:115` (strict decode — guard point), `internal/mcp/executor_client.go:50-56,110-145,161-186,387-411` (transport construction, `authorizationRoundTripper`, WWW-Authenticate handling, secret resolution/redaction), `internal/mcp/hosted_proxy.go` (agent→remote path), `internal/session/manager_start_mcp.go` (assembly), `internal/cli/extension_secrets.go:120-154` (bind verb), `internal/api/core/extensions_secrets.go` + `contract/extensions.go:94-98` (secrets contract), `internal/extensionenv/bindings.go` (binding types), `internal/api/spec/registry_extensions.go` (OpenAPI registry).

Reference implementations:

- Fail-closed transport-capability rule for an adapter-invented flag: `.resources/hermes/tools/mcp_tool.py:3030-3086` (`strict_redirect_headers`)
- No host-env interpolation into adapted configs (asymmetry): `.resources/hermes/tools/mcp_tool.py:5124-5142`
- Real-subprocess env round-trip E2E fixture: `.resources/codex/codex-rs/core/tests/suite/plugins.rs:686-760`

Skills to activate: `eng-code-guidelines`, `golang-master`, `eng-test-conventions`, `testing-boss`, `eng-cleanup-failure-paths`, `eng-contract-codegen-coship`, `eng-consolidate-test-suites`.

### Relevant Files

- `internal/mcp/executor_client.go` — the seam everything header-related hangs off
- `internal/config/provider.go:181-189` — `MCPAuthConfig` (OAuth interplay input to the policy)
- `internal/session/manager_lifecycle.go:143-171` — merge path the projections ride
- `internal/daemon/extension_secrets.go` + secret hygiene integration tests — redaction suites to extend

### Dependent Files

- `internal/api/contract/extensions.go` — secrets payload widening (this task) + extension payload fields (task_04)
- `web/src/generated/compozy-openapi.d.ts` — regenerates; consumed by task_05
- `internal/cli/extension_output.go` — task_04 renders `secrets list` mapping suffix

### Related ADRs

- [ADR-006: MCP headers daemon-side](adrs/adr-006.md) — the contract this task implements end-to-end
- [ADR-003: First-class diagnostics](adrs/adr-003.md) — live-health projection obligations

### Web/Docs Impact

- `web/`: `web/src/generated/compozy-openapi.d.ts` regenerates from the secrets contract widening (no component work here — task_05 owns rendering; checked: no other web surface).
- `packages/site`: none — `extensions/secrets.mdx` update lands in task_06.
- QA impact: new behavior (authenticated remote server via binding; live unhealthy diagnostics) — add content-addressed `untested` scenario files (added in task_06's sweep; walked by task_08).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: MCP-sidecar file format explicitly unaffected (closed-surface guard is the evidence); extension manifest `MCPServerConfig` consumption from task_02 unchanged here; no hook/provide/registry change (checked: surface registry, `extensionprotocol`).
- Agent manageability: secrets binding operable via structured CLI (`-o json`) + HTTP/UDS with identical deterministic errors; **deliberately no native secrets tool** (existing zero-secrets-tool posture, documented); health visible to agents via the diagnostics fields task_04 projects.
- Config lifecycle: zero `config.toml` keys (checked: `ExtensionsConfig`, marketplace config); OpenAPI/codegen co-ship (`make codegen`, `make codegen-check`, `make bun-typecheck` listed under Tests per contract co-ship).

## Deliverables

- Headers/secret-references flowing end-to-end: package headers + bound secrets reach the remote server from the daemon; nothing secret ever serializes
- Runtime-health registry live and projected; data-dir first-launch creation
- Secrets remote-header vertical on all three paths + regenerated contracts
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**
- `make gate` green; `make codegen-check` clean

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-034, UT-035 — widened server validation arms + clone/reference-shape discipline
- [x] UT-037 — sidecar closed-surface rejection
- [x] UT-046 — `--remote-header` flag parse table
- [x] UT-052 — secret-reference serialization negatives (clone/JSON/YAML/TOML/reads)
- [x] IT-006 — executor headers + connection-time resolution + redaction + live-health lifecycle (clear/ordering/evict/restart)
- [x] IT-011 — binding parity across CLI/HTTP/UDS, dangling ref, presence-only reads
- [x] IT-017 — first-launch data-dir creation failure → live health for that server only
- [x] E2E-001 — runtime distribution journey (install → enable → acpmock session skill + stdio env round-trip + data-dir-at-first-launch; 8-item checklist walk)

## Success Criteria

- Every assigned test case implemented and passing
- A resolved secret value is grep-provably absent from every serialized form, log, status read, and provider projection exercised by the suites
- An injected out-of-order stale failure never overwrites a newer success; reload/remove evict health entries
- E2E-001's checklist walk records the conformance evidence the listing PR (task_09) will cite
