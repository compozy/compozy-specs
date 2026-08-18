---
status: completed
title: Conformance foundations: mcppolicy + agentplugin
type: backend
complexity: high
---

# Task 1: Conformance foundations: mcppolicy + agentplugin

## Overview

Delivers the two new leaf packages every later task builds on: `internal/mcppolicy` (the single source-aware header/URL policy authority) and `internal/extension/agentplugin` (the Agent Plugins 1.0.0 conformance reader — `$schema` triage, fatality ladder, non-recursive skill discovery, MCP entry translation, placeholder expansion, scoped diagnostics). Both are pure, exhaustively unit-tested, and carry zero daemon wiring, so this task is the dependency contract for the ingestion core.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST create `internal/mcppolicy` (stdlib-only leaf) implementing `ValidateHeaders(fixed, secret, source, authEnabled)` and `ValidateRemoteURL` exactly per `_spec.md` Part II Core Interfaces: RFC 7230 token names, CR/LF-free values, case-insensitive duplicate rejection across the union of both maps, `content-type`/`mcp-*` always rejected, sensitive set (`authorization`, `proxy-authorization`, `cookie`, `set-cookie`) rejected for `SourcePackageFixed` with a stable scoped code, operator `authorization` allowed iff no OAuth, URL policy (absolute http/https, no userinfo/fragment, plain http loopback-only).
2. MUST create `internal/extension/agentplugin` with the closed import list (stdlib + `internal/frontmatter` + `internal/mcppolicy`) and the fixed production file split: `types.go`, `classify.go`, `manifest.go`, `skills.go`, `mcp.go`, `paths.go` — each under the 500-line cap from the first commit.
3. MUST implement the tri-state `$schema` triage (`SchemaSupported` / `SchemaUnsupportedVersion` / `SchemaUnrelated`), the fatality ladder (fatal manifest errors as `*ManifestError` with issues; component problems as scoped `Diagnostic`s that never fail the load), and `ValidateName` as RE2-safe split checks (no lookahead — length bound, byte-level charset, alnum edges, `--`/`..` contains-checks).
4. MUST enforce the standard's normative rules: closed manifest schema with report-and-ignore for unknown top-level fields, explicit-null-is-fatal vs absent-is-fine (raw JSON pre-pass), typed `AuthorInfo`, non-recursive immediate-children skill discovery with agentskills frontmatter conformance (name==dir, description bounds), MCP transport union with validate-then-reject `sse` (two distinct messages), command-as-single-token, cwd normalization (omitted → package root; `./`, `${PLUGIN_ROOT}`, `${PLUGIN_DATA}` prefixes with two canonical containment domains), reserved-env rejection, single-pass non-recursive placeholder expansion applied only to args/env-values/cwd, and the cross-document `mcp.json`↔`plugin.json` schema-version match (mismatch disables MCP only).
5. MUST re-download both standard schemas byte-for-byte once, pin the canonical identifiers as constants, and record the URLs + content digests in this task's completion notes (no runtime fetching — `_spec.md` Technical Dependencies).
6. MUST keep `Load` deterministic: identical package bytes + identical canonical inputs (root, data dir, instance key) → identical output; sorted directory walks.
7. MUST NOT import `internal/extension`, `internal/config`, `internal/skills`, or `internal/api/contract` from `agentplugin`; MUST add the `mcppolicy` leaf rule and the `agentplugin` closed-import rule to `magefiles/boundaries.go` in the same change.
</requirements>

## Subtasks

- [x] 1.1 Re-download `plugin.schema.json` + `mcp.schema.json` byte-for-byte; pin identifiers/digests; record in completion notes
- [x] 1.2 Create `internal/mcppolicy` (policy types, `HeaderSource`, `ValidateHeaders`, `ValidateRemoteURL`) with its unit table
- [x] 1.3 Create `agentplugin` contracts (`types.go`): `Package`, `AuthorInfo`, `SkillRef`, `ServerSpec`, `Diagnostic`, `ManifestError`, `LoadOptions`
- [x] 1.4 Implement `classify.go` ($schema triage incl. symlink/non-regular refusal) and `manifest.go` (fatality ladder, name validation, null-vs-absent pre-pass, unknown-field pop + diagnostic)
- [x] 1.5 Implement `skills.go` (immediate-children discovery via `internal/frontmatter`, agentskills conformance checks, per-skill skip diagnostics)
- [x] 1.6 Implement `mcp.go` (mcp.json shape, cross-doc version rule, transport union, per-entry isolation `{servers, errors}` outcome, sse dual-message, header/URL policy via `mcppolicy`)
- [x] 1.7 Implement `paths.go` (dual-domain containment, command-token rule, cwd normalization, single-pass expansion, reserved env injection-last)
- [x] 1.8 Update `magefiles/boundaries.go` with both package rules
- [x] 1.9 Implement the full assigned unit suite, including the adversarial fixtures (self-referential `${PLUGIN_DATA}` root path, 8-bad-1-good isolation set, real symlink escapes)

## Implementation Details

Follow `_spec.md` Part II → Core Interfaces (signatures are final) and System Architecture (file split). The reference implementations are load-bearing reading — steal the patterns, not the code:

- Name predicate without lookahead: `.resources/codex/codex-rs/core-plugins/src/agent_plugin_manifest.rs:219-235`
- Tri-state `$schema` triage: `.resources/codex/codex-rs/utils/plugins/src/plugin_namespace.rs:17-64`
- Two-tier closed schema (top-level allowlist+warn+strip; nested closed) and explicit-null fatal: `.resources/codex/codex-rs/core-plugins/src/agent_plugin_manifest.rs:75-127`
- Per-entry isolation as an explicit return type + reserved env + cwd containment + single-pass expansion: `.resources/codex/codex-rs/codex-mcp/src/agent_plugin_config.rs:97-108,177-186,349-396`
- Depth-capped skill discovery: `.resources/codex/codex-rs/ext/skills/src/loader/discovery.rs:69-72`
- Two error classes with scoped diagnostics + validate-then-reject sse + raw-pattern reference: `.resources/hermes/hermes_cli/agent_plugins.py:39-54,504-518`
- Adversarial test fixtures: `.resources/codex/codex-rs/codex-mcp/src/plugin_config_tests.rs` and `.resources/hermes/tests/hermes_cli/test_agent_plugins.py:19-40`

Skills to activate: `eng-code-guidelines`, `golang-master`, `eng-test-conventions`, `testing-boss`, `eng-consolidate-test-suites`.

### Relevant Files

- `internal/frontmatter/` — existing YAML frontmatter parser `agentplugin` is allowed to import
- `internal/diagnosticcontract/diagnostics.go` — canonical `DiagnosticItem` shape the converters (task_02) will target; do NOT import from `agentplugin`
- `magefiles/boundaries.go` — CI-enforced import rules; gains both new package rules
- `.compozy/tasks/agent-plugins/analysis/analysis_agent_plugins_standard.md` — verbatim schemas, fatality rules, §9 runtime semantics, 8-item checklist

### Dependent Files

- `internal/extension/manifest_load.go` — task_02 branches into `agentplugin` from here
- `internal/registry/installer_metadata.go` — task_02 uses `ClassifyManifest` for root triage
- `internal/config/mcp_server_validation.go` — task_03 calls `mcppolicy` from the config validation arms

### Related ADRs

- [ADR-001: Third install layout](adrs/adr-001.md) — the package this task founds
- [ADR-004: Verbatim names](adrs/adr-004.md) — `ValidateName` is the enforcement point
- [ADR-006: MCP headers](adrs/adr-006.md) — `mcppolicy` is the single policy authority it mandates

### Web/Docs Impact

- `web/`: none — checked surfaces: no contract, route, or generated-type change in this task; both packages are pure leaves.
- `packages/site`: none — docs land in task_06.
- QA impact: none — no user-visible behavior change (pure packages; behavior surfaces in tasks 02–05).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: founds the conformance layer ADR-001 requires; no hook/provide/permission/bridge/MCP-sidecar surface changes — checked: `capabilities.go`, `host_api_methods_gen.go`, bridge contract untouched.
- Agent manageability: none — no CLI/HTTP/UDS surface in this task (arrives in task_04).
- Config lifecycle: none — zero config keys; schema identifiers are compile-time constants (checked: `ExtensionsConfig` untouched).

## Deliverables

- `internal/mcppolicy` + `internal/extension/agentplugin` complete, boundary-enforced, each file under the 500-line cap
- Pinned schema identifiers + recorded digests in completion notes
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**
- `make gate` green for the affected lanes

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-001, UT-002, UT-003, UT-004 — `$schema` triage + non-regular manifest refusal
- [x] UT-005, UT-006 — name validation accept/reject tables
- [x] UT-007, UT-008, UT-009, UT-010 — manifest fatality ladder, null-vs-absent, unknown-field pop, extensions namespaces
- [x] UT-011, UT-012, UT-013, UT-014 — skill discovery (non-recursive, name==dir, bad frontmatter, skills-as-file)
- [x] UT-015, UT-016, UT-017, UT-018, UT-019, UT-020, UT-021, UT-022, UT-023, UT-024 — mcp.json translation, whole-file failures, sse dual-message, isolation set, command token, cwd containment, reserved env, expansion, headers, URL policy
- [x] UT-025, UT-026 — empty package, adversarial/duplicate server keys
- [x] UT-036 — `mcppolicy.ValidateHeaders` source-aware matrix
- [x] UT-055 — cross-document schema-version mismatch disables MCP only
- [x] UT-056 — cwd normalization defaults + data-root domain + per-domain symlink escapes

## Success Criteria

- Every assigned test case implemented and passing
- `mage Boundaries` enforces both new package rules; no forbidden import compiles
- The 8-item minimum-conformance checklist items owned by the reader (§11.1 items 2, 3, 4 partial, 5 partial, 7 rules) are demonstrably covered by the suite
- Deterministic `Load` proven by the repeat-load deep-equal case

## Completion Notes

- Pinned authoring inputs (downloaded byte-for-byte on 2026-08-15; no runtime fetch):
  - `https://agent-plugins.org/schemas/1.0.0/plugin.schema.json` — SHA-256 `0a4aad95ce337878ad38802ebf0daa3fde76abe3f65400c86bcbb1ec0b3ab883`
  - `https://agent-plugins.org/schemas/1.0.0/mcp.schema.json` — SHA-256 `6539175bfcdf43085855183e86da40ea94b166547a72b47ae9a0a390516d3acb`
- Focused evidence: `CGO_ENABLED=1 go test -race` passed for `internal/mcppolicy`, `internal/extension/agentplugin`, and `internal/frontmatter`; coverage was 88.6% and 81.4% for the two new packages; scoped Go lint reported zero issues; test-shape checks, `boundaries`, and `git diff --check` passed.
- The task-local `make gate` deliverable is intentionally deferred under the accepted loop directive: no broad Make/gate commands before the final QA/review tail.
- QA tracker: no scenario flag — this task creates pure, unwired leaf packages and changes no user-visible behavior.

Compozy Impact Audit:

- Native tools: no impact — checked native tool IDs/toolsets/descriptors/schema digests and capability gates; neither leaf is wired into daemon or tool registration in this task.
- Extensibility and hooks: foundational-only — added the closed-import conformance and MCP policy leaves; checked extension registries, hooks, capabilities, bridge SDKs, MCP sidecars, and `config.toml`; no public registration, hook, sidecar, or config lifecycle changes occur until downstream tasks.
- Workspace data isolation: no runtime datum introduced — `LoadOptions.DataDir` is an explicit instance input used only for pure path normalization; no global/workspace/session/agent store, cache, SSE, or event path is read or written.
- Official Compozy skill: no impact — checked `skills/compozy/`; no public CLI/API/tool/hook/capability behavior is wired in this task.
