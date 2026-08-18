# Test Specification: Extension DX Overhaul (ext-improvs)

Canonical test contract for the extension DX program. Companion to `_techspec.md`. Derived from `_brief.md` requirements R1–R11 (no `_user_stories.md` exists for this program — journeys are derived from the brief's Verification section; coverage gap noted: no persona-level story catalog, so E2E journeys bind directly to requirements).

## Strategy

- Frameworks: Go `testing` with `-race` (+`+integration` tag where marked), table-driven `t.Run("Should …")` + `t.Parallel` default (no `t.Parallel` on env-mutating cases per L-002); Vitest for `sdk/typescript`; Playwright for web E2E; the Go E2E runtime harness for daemon-side journeys.
- Fakes at I/O boundaries only: mocked GitHub API server, stub `git` runner, `MockTransport`/describe-mode fixtures for subprocess; real SQLite, real registry, real daemon in integration/E2E.
- Execution: unit `go test -race ./internal/<pkg>/...` scoped per package + `bunx turbo run test --filter=./sdk/typescript`; integration `make test-integration`; runtime journeys `make test-e2e-runtime`; web `make test-e2e-web`; contract drift `make codegen-check`; full gate `make verify` at completion.
- Cross-build gate (not an ID'd case): `GOOS=windows GOARCH=amd64 go build ./...` must pass for subprocess/dev-lane changes.
- Migration cases extend the canonical fresh/reopen/ahead/integrity/equivalence suites owned by `internal/store` (no parallel suite).
- Every case group is assigned to its owning canonical suite in the Suite Placement section below (`eng-consolidate-test-suites`); implementers extend the named suite — standalone duplicates are rejected in review.

## Suite Placement

Owning suite per case group. "(existing)" = file verified present today; "extends" = add table-driven subtests to that package's suite for the named source file; "new suite" = no owner exists for a new component, with the rationale being the component itself is new.

| Case group | Owning suite | Status |
| --- | --- | --- |
| UT-001–UT-005 manifest v2 + closed sets | `internal/extension/manifest_test.go` (existing, verified — `manifest_load_test.go` does not exist) | extends |
| UT-006–UT-007 consent derivation | `internal/extension/capability_test.go` (existing, verified) | extends |
| UT-008–UT-014 build/validate | `internal/extension/build_test.go` | new suite (new `build.go` component) |
| UT-015–UT-021, UT-064–UT-066, UT-086 dev links + coordinator + generation handle | `internal/extension/dev_test.go` | new suite (new `dev.go` component) |
| UT-022–UT-025, UT-068–UT-069 trust gates | `internal/extension/marketplace_lifecycle_test.go` (existing, verified — `marketplace_trust_test.go` does not exist; this suite owns the install/trust lifecycle) | extends |
| UT-026–UT-030 source-union parsing | `internal/cli/extension_install_parse_test.go` — the parser is CLI-owned by decision: shorthand refs exist only on the CLI surface, while HTTP/UDS/native callers submit the structured union directly | new suite (new parser) |
| UT-031–UT-032 gitsrc | `internal/registry/gitsrc/client_test.go` | new suite (new package) |
| UT-033–UT-034 publish | `internal/extension/publish_test.go` | new suite (new `publish.go` component) |
| UT-035–UT-036 hook tier + ordering | `internal/hooks/ordering_test.go` (existing, verified) | extends |
| UT-037 extension hook stamping | `internal/extension/manager_test.go` (existing, verified — owns manager resource loading) | extends |
| UT-038–UT-039 generated Go contracts | `internal/codegen/sdkgo/generate_test.go` (mirrors `internal/codegen/sdkts/generate_test.go`, verified) | new suite (new generator) |
| UT-040, UT-070 check-mode drift | `cmd/compozy-codegen/main_test.go` (existing, verified) | extends |
| UT-041–UT-045 SDK Go | `sdk/go/extension_test.go` + `sdk/go/runtime_contract_test.go` (both existing, verified; UT-045 extends the boundary test) | extends |
| UT-046–UT-049 SDK TS | `sdk/typescript/src/__tests__/extension.test.ts` (existing, verified; harness-based) | extends |
| UT-050–UT-051 human error rendering | `internal/cli/root_test.go` | new suite (`renderHumanExecutionError` is new; `root.go` has no owning suite today) |
| UT-052–UT-055 CLI bundles + golden output | `internal/cli/extension_output_test.go` | new suite (`extension_output.go` has no owning suite today) |
| UT-056–UT-057 config defaults + validation | `internal/config/config_test.go` (existing, verified) | extends |
| UT-058 overlay merge | `internal/config/merge_test.go` (existing, verified) | extends |
| UT-059 `config set` key registry | `internal/cli/config_test.go` (existing, verified) | extends |
| UT-060 doctor probe | `internal/api/core/extension_doctor_test.go` (per-probe pattern of `bridge_doctor_test.go`/`provider_doctor_test.go`; `doctor_payload_test.go` does not exist) | new suite (new probe) |
| UT-061 event coverage matrix | `internal/extension/events_coverage_test.go` — call-site event-sink matrix for the new extension lifecycle event set (no extension event matrix exists today; emission is asserted at the owning layer, not via log scraping) | new suite (new event set) |
| UT-062, UT-067 logs ring + redaction | `internal/extension/logs_test.go` | new suite (new ring-buffer component) |
| UT-063 status-code mapping | `internal/api/core/extensions_test.go` (existing, verified) — extends the `ExtensionStatusCode` table | extends |
| UT-071–UT-075, UT-080, UT-082–UT-083, UT-085 command spec + groups + build rejections | `internal/extension/command_test.go` | new suite (new `command.go` component) |
| UT-076–UT-079, UT-081, UT-084 argv→input + rendering | `internal/cli/extension_exec_test.go` (parse/convert + golden output) | new suite (new `exec` verb) |
| IT-001 migrations | `internal/store/migrate_test.go` + `internal/store/migrate_streams_test.go` (both existing, verified — the fresh/reopen/ahead/integrity/equivalence subtests covering the global catalog stream; no parallel suite) | extends |
| IT-002–IT-004, IT-007 install pipelines + batch update | `internal/extension/marketplace_lifecycle_test.go` (existing, verified — owns install/update lifecycle incl. `UpdateBatch`) | extends |
| IT-005 describe-mode round trip | `internal/extension/build_integration_test.go` | new suite (new build component) |
| IT-006, IT-015, IT-017 dev loop + workspace isolation + coordinator barrier | `internal/extension/dev_integration_test.go` | new suite (new dev component) |
| IT-008 search union | `internal/daemon/extension_search_integration_test.go` (the union composes in the daemon service layer) | new suite (new composition) |
| IT-009 logs route + SSE follow | `internal/api/httpapi/extension_logs_sse_test.go` (named-event SSE contract is transport-specific) | new suite (new route) |
| IT-010 HTTP↔UDS parity | `internal/api/core/extensions_test.go` (existing, verified — parity pattern of `settings_test.go`/`automation_test.go`) | extends |
| IT-011 native tools | `internal/daemon/native_extension_tools_integration_test.go` (existing, verified) | extends |
| IT-012 hook introspection | `internal/daemon/hook_binding_resources_test.go` (existing, verified) | extends |
| IT-013 public harnesses | `sdk/go/extensiontest/harness_test.go` (new public package) + `sdk/typescript/src/__tests__/integration.test.ts` (existing, verified) | new + extends |
| IT-014 end-to-end drift gate | `cmd/compozy-codegen/main_test.go` (existing, verified) | extends |
| IT-016 secret-transport absence | `internal/daemon/extension_secret_hygiene_integration_test.go` (cross-transport matrix owned at the composition root) | new suite (new matrix) |
| IT-018 commands read-model parity | `internal/api/core/extensions_test.go` (existing, verified) | extends |
| IT-019 exec single-invoke | `internal/cli/extension_exec_integration_test.go` (transport spy on the CLI client) | new suite (new verb) |
| IT-020–IT-021 policy parity + nesting | `internal/daemon/extension_commands_integration_test.go` | new suite (new read model) |
| E2E-001, E2E-007 authoring + docs-verbatim | `internal/daemon/daemon_extension_authoring_e2e_integration_test.go` (`TestDaemonE2E…` runtime-lane pattern, verified against existing `daemon_*_e2e_integration_test.go` journeys) | new journey file |
| E2E-002 distribution | `internal/daemon/daemon_extension_distribution_e2e_integration_test.go` | new journey file |
| E2E-003 agent native-tool journey | `internal/daemon/daemon_extension_agent_native_e2e_integration_test.go` | new journey file |
| E2E-004 error UX golden | `internal/daemon/daemon_extension_error_ux_e2e_integration_test.go` | new journey file |
| E2E-008 contributed-command journey | `internal/daemon/daemon_extension_commands_e2e_integration_test.go` | new journey file |
| E2E-005 marketplace update affordance | `web/e2e/__tests__/marketplace.spec.ts` (existing, verified) | extends |
| E2E-006 extensions dev badge/logs/install | `web/e2e/__tests__/extensions.spec.ts` (sibling of `marketplace.spec.ts`) | new spec (new surface) |

## Coverage Matrix

| Source | Behavior | Unit | Integration | E2E |
| --- | --- | --- | --- | --- |
| R1 (time-to-first-extension) | scaffold → build → validate first success | UT-008–UT-014 | IT-005 | E2E-001, E2E-007 |
| R2 (own-code install trivial) | dev lane, no trust ceremony, replace-in-place | UT-015–UT-022 | IT-006 | E2E-001 |
| R2 (published sources) | source union parse + install per source; integrity never elevates trust | UT-026–UT-032, UT-068 | IT-002–IT-004 | E2E-002 |
| R3 (updates announce themselves) | update_available projection + batch progress | UT-053 | IT-007 | E2E-005 |
| R4 (inner loop) | reload, watch, logs, crash-loop honesty | UT-019–UT-021, UT-054, UT-062 | IT-006, IT-009 | E2E-001, E2E-006 |
| R5 (SDK simple / single declaration) | describe mode, manifest generation, closed sets | UT-001–UT-014, UT-041, UT-046 | IT-005 | E2E-001 |
| R6 (SDK complete) | trusted_workspace, clarify, required-methods, 4 public provides + bridge gate | UT-039, UT-042–UT-043, UT-047–UT-049, UT-069 | IT-013 | E2E-003 |
| R7 (SDK current) | generated Go/TS contracts + CI drift gate | UT-038, UT-040, UT-045 | IT-014 | — |
| R8 (errors help, output confirms) | human diagnostics, success hints, status summary | UT-050–UT-052, UT-054–UT-055, UT-063 | — | E2E-004 |
| R9 (concept bill / config) | config consolidation lifecycle | UT-056–UT-059 | — | — |
| R10 (no-gatekeeper publish; agents first-class) | publish verb, search union, native tools | UT-033–UT-034 | IT-008, IT-011 | E2E-002, E2E-003 |
| R11 (prerequisites) | version-gate CI key, phantom capabilities dead, hook source | UT-002–UT-005, UT-012, UT-035–UT-037 | IT-001, IT-012 | E2E-007 |
| Component: registry schema v2 + dev-links table | migration hard cut | — | IT-001 | — |
| PR-B003 (workspace boundary) | dev/reload/logs workspace authorization + path containment | UT-064–UT-066 | IT-015 | — |
| PR-B005 (secret hygiene) | server-side credentials + ingestion-time redaction | UT-033–UT-034, UT-067 | IT-016 | — |
| PR-B006 (atomic generations) | coordinator serialization + last-good activation | UT-020–UT-021 | IT-017 | — |
| PR-B007 (describe single-source) | handshake drift gates both SDKs | UT-070 | IT-014 | — |
| PR2-B001 (instance-keyed lifecycle) | per-workspace instances for reload/logs/status/events | UT-064–UT-066 | IT-015 | — |
| PR2-B002 (generation handle) | hash-only generation identity + re-validation before activation | UT-065, UT-086 | IT-015–IT-016 | — |
| PR2-B005 (declared groups path) | groups through describe → manifest → read model | UT-082–UT-083 | IT-018 | E2E-008 |
| PR2-B006 (closed flag subset) | accepted/rejected projection matrix + conversion semantics | UT-084–UT-085 | — | — |
| ADR-008 (contributed commands) | command metadata, `/`-nested paths, flag projection, exec-through-tools | UT-071–UT-085 | IT-018–IT-021 | E2E-008 |
| Component: HTTP/UDS parity | new routes both transports | — | IT-010 | — |
| Component: doctor/observability | probe + event coverage | UT-060–UT-061 | — | — |
| Web surfaces | update affordance, dev badge, logs panel | — | — | E2E-005, E2E-006 |

## Unit Tests

### Manifest v2 + closed sets (TechSpec: Implementation Design / Data Models)

- **UT-001** (happy): `LoadManifest` on a v2 bundle with `[permissions] requires = ["sessions/list"]` — populates `Manifest.Permissions.Requires` exactly; no `Actions`/`Security` fields exist.
- **UT-002** (error): manifest with `capabilities.provides = ["prompt.provider"]` — load fails with an error naming the value and listing the five valid provides.
- **UT-003** (error): `permissions.requires = ["sessions/does-not-exist"]` — load fails naming the unknown method.
- **UT-004** (error): manifest containing a legacy `[actions]` or `[security]` section — load fails with an error naming `[permissions]` as the replacement.
- **UT-005** (boundary): `min_compozy_version` one patch above the stamped daemon version — `ManifestCompatibilityError`; equal version loads.
- **UT-006** (happy): `DeriveConsentAreas(["sessions/list","memory/store"])` returns `[{sessions,read},{memory,write}]` from the generated table.
- **UT-007** (error): `DeriveConsentAreas(["nope/x"])` returns a membership error (never an empty silent result).

### Build / validate (TechSpec: Core Interfaces — BuildBundle/ValidateBundle)

- **UT-008** (happy): `BuildBundle` toolchain detection — `package.json` with `build` script selects it; `go.mod` selects `go build -o dist/bin`; `BuildCmd` override wins (stub runner records argv).
- **UT-009** (idempotency): two `BuildBundle` runs over identical source produce byte-identical `extension.toml` (invariant 11).
- **UT-010** (error): describe-mode process exceeding `Timeout` — build fails naming the timeout; no partial `dist/` manifest left.
- **UT-011** (error): describe payload carrying an unknown provide — build fails before writing any manifest.
- **UT-012** (happy): generated manifest `min_compozy_version` equals `DescribeSDKInfo.MinCompozyVersion` — authored values in source are ignored.
- **UT-013** (happy): `ValidateBundle` on TOML with a syntax error at line 7 col 3 — `ValidationIssue{Path,Line:7,Column:3}`.
- **UT-014** (happy): `ValidateBundle` on a valid bundle returns the derived consent areas in the payload.

### Dev lane + registry (TechSpec: Core Interfaces — DevLink/ReloadExtension; Safety Invariants 1–3, 7–8)

- **UT-015** (happy): `LinkDev` upserts an `extension_dev_links` row `(name, workspace_id, origin_path, bundle_generation)`; `ResolveActive` prefers it; the published `extensions` row is byte-identical before/after.
- **UT-016** (state): `LinkDev` twice with the same `(name, workspace_id)` — second upserts in place; no `ExtensionExistsError`.
- **UT-017** (state): `LinkDev` over a published row sets `overrides_published=true` in the projection; `UnlinkDev` deletes only the side-table row and `ResolveActive` returns the untouched published row.
- **UT-018** (error): every `install` path with any request shape — no `extension_dev_links` row is ever created (invariant 1 table-driven over all source kinds).
- **UT-019** (error): dev link whose `origin_path` no longer exists — status `errored`/`missing_origin`; `Manager.Start` returns nil (no boot crash), subprocess never spawned.
- **UT-020** (concurrency): 10 goroutines interleaving `ReloadExtension(ctx, InstanceKey{"x", wsA}, <freshly built GenerationHash>)` with a concurrent build staging a new generation — the per-instance operation coordinator serializes; observable states are only (old generation running) → (new generation running); a parallel reload on `InstanceKey{"x", wsB}` proceeds independently on its own coordinator; race-detector clean; no torn/partially-read bundle (invariant 7, barrier-controlled).
- **UT-021** (state): reload whose new generation fails activation — the **last-good generation is restarted and keeps serving**; status reports `errored (activation_failed; running <prior generation>)` with `last_error` set (invariant 8).
- **UT-064** (error): hostile workspace input — a request carrying a forged `workspace_id` for dev/reload/logs is ignored: the handler binds the identity from the authenticated caller scope, and a session bound to workspace A can neither list, reload, nor read logs of workspace B's dev link; the same binding table proves the global `(name, "")` instance's logs succeed through operator transports and are denied for agent callers (invariant 2, table-driven across HTTP/UDS/native bindings).
- **UT-065** (error): symlink escape — an `origin_path` whose resolved target escapes the bound workspace root (including the macOS `/private/var` canonicalization quirk) is rejected at link time and at load time (invariant 3).
- **UT-066** (happy): operator CLI `dev` binds `workspace_id` from the resolved current workspace; the link row records it and agent projections for that workspace see the extension.
- **UT-086** (error): generation-handle hostility — a malformed hash (path separators, traversal sequences, non-hex), a hash naming no generation under the canonicalized origin, and a stale generation whose re-verified manifest/digest no longer match — link and reload each reject deterministically (`ErrExtensionGenerationInvalid`), and no request shape can cause execution outside `<origin>/dist/gen-<hash>` (invariant 3).

### Trust gates (TechSpec: Safety Invariants 4–5; ADR-002/ADR-005)

- **UT-022** (happy): dev-lane link performs zero marketplace-trust evaluations (spy resolver never called).
- **UT-023** (happy): unverified side-load with `allow_unverified=true` request under default policy (`trust.allow_unverified=true`) installs with provenance `allowed_unverified`.
- **UT-024** (error): policy `false` + consent flag — `ExtensionPolicyBlockedError` whose diagnostic carries `compozy config set extensions.trust.allow_unverified true`.
- **UT-025** (error): digest sidecar mismatch — `ExtensionArchiveDigestMismatchError`; registry row absent afterward (rollback, invariant 4); no consent flag bypasses a failed integrity check.
- **UT-068** (happy): github install **with matching sidecar** — provenance records `digest_matched: true` **and** `checksum_verified: false`, tier stays `unverified`, and the consent requirement is identical to the sidecar-absent case (invariant 5; a colocated digest never elevates trust).
- **UT-069** (error): installed manifest declaring `capabilities.provides = ["bridge.adapter"]` — deterministic "external bridge authoring is a planned follow-up" error (invariant 6 / ADR-006); the four public provides install normally.

### Install source union parsing (TechSpec: Data Models — InstallExtensionRequest)

- **UT-026** (happy): `github:acme/x@v1` → `{source: github, ref: acme/x@v1}`.
- **UT-027** (happy): bare `acme/x` → curated resolution first, github fallback second (order asserted via stub resolvers).
- **UT-028** (happy): absolute filesystem path → `{source: local_path}`.
- **UT-029** (error): `./typo-dir` that does not exist — path error naming the path; never degrades to a slug lookup.
- **UT-030** (happy): `git:https://example.com/r.git@v2` → `{source: git, ref: …@v2}`.

### gitsrc + publish (TechSpec: Integration Points; ADR-005)

- **UT-031** (happy): `gitsrc.Download` shallow-clones at the requested ref (stub git runner argv asserted) and returns an archive stream.
- **UT-032** (error): `git` binary absent — `ErrGitUnavailable` with a diagnostic advising installation.
- **UT-033** (happy): `PublishRelease` uploads archive + `.sha256` sidecar whose digest equals the freshly hashed archive; result carries release + asset URLs; the request struct has no credential field and the injected uploader credential appears in no error/event/log output.
- **UT-034** (error): no publish credential binding resolvable (daemon env/vault absent) — deterministic error naming the binding to configure (no API call attempted, no credential echoed).

### Hooks source (TechSpec: Core Interfaces — HookSourceExtension)

- **UT-035** (happy): `HookSourceExtension.String() == "extension"`; `DefaultHookPriority(HookSourceExtension) == 300`.
- **UT-036** (ordering): resolved hooks ordering config(500) > extension(300) > agent_definition(100) with stable tie-break.
- **UT-037** (happy): manager registers extension-declared hooks stamped `HookSourceExtension`, not `HookSourceConfig`.

### Generated contracts (TechSpec: codegen sdkgo; ADR-003)

- **UT-038** (happy): `sdkgo.Generate()` emits exactly the host-method and hook-event counts reported by `internal/extension/contract`'s `HostAPIMethodSpecs()`/`BuildHookContracts()` (counts compared, not hardcoded), plus the describe-schema representation and conformance fixtures.
- **UT-039** (happy): generated `RequiredMethods("bridge.adapter")` contains `bridges/deliver` **and** `bridges/targets/snapshot`; `RequiredMethods("model.source")` contains `models/list`.
- **UT-040** (error): in a **temporary copy of the generated tree** (never the working tree), mutating one byte of a generated Go contract file — `compozy-codegen check` against that path returns `ErrStaleGeneratedFile` naming it.
- **UT-070** (error): describe-schema drift — regenerating fixtures after a mutated `internal/extension/contract` describe spec fails `codegen-check` for **both** SDK representations (the handshake cannot silently diverge; invariant 10).

### SDK Go (TechSpec: sdk/go layers)

- **UT-041** (happy): `Tool[SearchInput]` registration → describe payload includes the tool descriptor with computed schema digests.
- **UT-042** (happy): a `tools/call` carrying `trusted_workspace` — handler's `ToolRequest` exposes the workspace scope verbatim (invariant 9).
- **UT-043** (happy): `AskClarification` round-trips `invocation_id` (unexported, unforgeable — existing property retained).
- **UT-044** (error): `initialize` granting fewer permissions than the definition requires — `capability_denied` naming the missing methods.
- **UT-045** (happy): `sdk/go` compiles as a standalone module and imports nothing under `internal/` (existing `TestSDKHasNoDaemonInternalImports` extended to the module boundary).

### SDK TypeScript (TechSpec: sdk/typescript)

- **UT-046** (happy): `__describe` argv makes the extension print a valid `DescribePayload` and exit 0 (TestHarness).
- **UT-047** (happy): tool handler context exposes `trustedWorkspace` and `invocationId` parsed from the request.
- **UT-048** (happy): `clarify/ask` callable from a TS tool handler — `invocation_id` attached automatically.
- **UT-049** (happy): initialize validation uses the generated required-methods map — a `model.source` extension missing `models/list` fails initialize.

### CLI output + errors (TechSpec: Agent Manageability; R8)

- **UT-050** (happy): `renderHumanExecutionError` on the policy-blocked error prints the message, the remediation sentence, and `try: compozy config set extensions.trust.allow_unverified true`.
- **UT-051** (happy): daemon-unreachable error human output contains the `compozy daemon start` suggestion (golden).
- **UT-052** (happy): `-o jsonl` on `status`, `remove`, and `provenance` emits one valid JSON object per line (no "jsonl formatter is required" error).
- **UT-053** (happy): `extension list` human table renders an `Update` column showing `→ 0.2.0` when `update_available` is set.
- **UT-054** (happy): `status` human output includes `consecutive_failures`, `restart_backoff`, and a one-line summary (`crash-looping (4 failures, backoff 8s)`) derived from the four axes.
- **UT-055** (happy): successful `dev`/`install`/`reload` print a `✓ <verb> <name>` line plus a `next:` hint (golden).

### Config lifecycle (TechSpec: Config Lifecycle; ADR-007)

- **UT-056** (happy): defaults — `extensions.trust.allow_unverified=true`, `extensions.sources.github.enabled=true`, `base_url=https://api.github.com`, `extensions.dev.watch_interval=2s`.
- **UT-057** (error): TOML containing `extensions.marketplace.allow_unverified` — validation error naming `extensions.trust.allow_unverified` as the replacement.
- **UT-058** (happy): overlay merge — workspace-level `[extensions.sources.github] enabled=false` overrides global true.
- **UT-059** (happy): `compozy config set` registry accepts every new `[extensions.*]` key and rejects the deleted ones with the replacement named.

### Doctor, observability, logs (TechSpec: Monitoring)

- **UT-060** (happy): doctor extension probe reports crash-loop (failures ≥ threshold), missing env, and stale dev origin as distinct diagnostic items.
- **UT-061** (happy): event coverage-matrix test asserts, through a test notifier/event sink registered at the emitting call sites, that **every** lifecycle path in the Monitoring set — `extension.{install,update,reload,publish}.{completed,failed}`, `extension.dev.{linked,unlinked}`, `extension.crash_loop.backoff` — emits its event with its per-event required keys from the Monitoring matrix (no over-broad or empty-valued keys) and with no secret value in any payload; removing any emission fails the matrix.
- **UT-062** (boundary): log ring buffer at 256 KiB — oldest lines dropped, writer never blocks (invariant 15; race-detector clean under concurrent writes).
- **UT-067** (happy): stderr ingestion redaction — a subprocess that echoes a resolved `secret_env` value and a provider token: the ring buffer, `Logs()` output, and the SSE frames all carry the masked form; the raw value appears in none (invariant 14, upstream-of-transport).

### Contributed commands (TechSpec: Core Interfaces — ExtensionCommandSpec/CommandDescriptor; ADR-008; invariants 16–17)

- **UT-071** (happy): build-time projection — a fixture tool declaring `command{verb:"greet", flags:{"count":"count","name":"name"}}` yields a `CommandDescriptor` whose flags carry the schema-derived types (`count` integer, `name` string) in canonical lexicographic order and whose `ToolID` is `ext__cmd_fixture__greet`; each flag materializes the full generated `CommandFlag` shape (type, repeatable, required, nullable, enum, default, bounds) exactly as the schema declares.
- **UT-072** (error): command declaring a reserved host flag (`--cmd`, `--input`, `--output`, `-o`, `--json`, `--help`) — build fails naming the flag and the reserved set (invariant 17).
- **UT-073** (error): flag mapped to a field absent from the input schema — build fails naming flag and field.
- **UT-074** (error): two tools in one extension declaring the same verb path — build fails naming the duplicate path; `review/fetch` vs a flat `review-fetch` both load but emit an ambiguity warning.
- **UT-075** (boundary): verb path depth — `greet` and `review/fetch` accepted; `review/round/fetch` rejected at build with the depth-2 rule stated (invariant 17).
- **UT-076** (happy): `buildCommandInput` argv→JSON — `--name my-feature --tasks-dir X` produces `{"name":"my-feature","tasks_dir":"X"}`; a repeated flag mapped to an array field accumulates; an absent optional flag is omitted (not null).
- **UT-077** (error): type conversion failure — `--round abc` for an integer field returns an error naming both the flag and the schema field.
- **UT-078** (error): `--input '{"name":"x"}'` combined with a projected flag — mutual-exclusion error; `--input` alone passes through validated against the schema.
- **UT-079** (error): unknown verb path — error lists the extension's available paths; `--cmd review` (a group) errors listing that group's leaves and never executes (invariant 17, groups non-executable).
- **UT-080** (happy): `Manager.Commands` projection — declared groups plus leaves, ordered deterministically; a disabled or unavailable extension contributes no commands.
- **UT-081** (happy): human-mode `extension commands` renders the group/leaf tree; `-o json` emits a flat path-keyed array with tool ids, flags, risk class, and approval metadata (golden).
- **UT-082** (error): group-declaration validation matrix — a group path equal to a leaf verb (leaf-as-parent), a group/leaf collision, an empty segment (`review//x`), leading/trailing slashes, duplicate group declarations, a multi-segment group path (`review/rounds`), and a declared group with no leaf under its prefix — each fails the build **and** manifest load naming the offending path (invariant 17).
- **UT-083** (happy): declared groups round-trip `DescribePayload.command_groups` → generated manifest `[[resources.command_groups]]` → `Manager.Commands` read model with their summaries; an undeclared prefix still yields an implicit group node (no summary) in the projection.
- **UT-084** (happy): projection conversion-semantics matrix — nullable scalar treated as the optional scalar; enum choices rendered in help with value validation left to invoke; numeric bounds rendered in help; boolean presence sets true and `--flag=false` sets false; a repeated flag on an array-of-scalar field accumulates; an absent optional flag omits the field (the CLI never injects schema defaults).
- **UT-085** (error): rejected-subset matrix — `object`, nested array, tuple/`prefixItems`, `oneOf`/`anyOf`/`allOf`/`not`, a multi-type union beyond nullable, and an unresolvable `$ref` on a mapped field — each fails the build **and** manifest load naming the field with the `--input` remediation (invariant 17).

### API contract mapping

- **UT-063** (error): `ExtensionStatusCode` — dev-origin-missing → 409; reload without an active dev link → 409 `ErrExtensionNotDevLinked`; unknown/malformed/stale generation hash → 400 `ErrExtensionGenerationInvalid`; install validation failure → 400 with `issues[]` payload; `bridge.adapter` install rejection → 400; git unavailable → 503-class; table-driven over the full error set.

## Integration Tests

### Registry schema + install pipelines

- **IT-001**: extensions-table v2 migration — fresh apply, reopen-with-data (pre-migration rows dropped per hard cut), ahead, and integrity cases appended to the canonical `internal/store` suites; `make codegen-check` passes with refreshed `atlas.sum`/sqlc output.
- **IT-002**: local-path install end to end on a real registry DB — consent flow, provenance `local_path`, auto-enable, `List` projection.
- **IT-003**: github install against a mocked GitHub API — release resolve → asset + `.sha256` download → integrity check → provenance `{installed_from: github, digest_matched: true, checksum_verified: false}`, tier `unverified`, consent required; sidecar-absent variant identical except `digest_matched: false`; mismatch variant aborts with rollback.
- **IT-004**: git install from a local fixture repository at a tag — provenance `git_url` populated (constant finally truthful).
- **IT-005**: describe-mode round trip — real Go SDK fixture subprocess → `BuildBundle` → generated manifest installs and activates; tool visible in the tool registry.

### Dev loop, updates, search, transport parity

- **IT-006**: `LinkDev` with a built generation's hash → the manager resolves `<origin>/dist/gen-<hash>`, re-verifies manifest + digest, and starts the subprocess → `CallTool` succeeds → rebuild fixture (new hash) → `ReloadExtension(ctx, key, newHash)` → new behavior observable; `UnlinkDev` restores the published copy.
- **IT-007**: `update --all` with two marketplace-installed extensions where the second fails — response reports per-item outcomes (first `updated`, second error) through HTTP and UDS (daemon `UpdateBatch` surfaced).
- **IT-008**: `GET /api/extensions/search?q=…&sources=curated,github` merges curated store + mocked topic search, each result tagged with source and verification badge; cursor pagination stable.
- **IT-009**: `GET /api/extensions/{name}/logs` returns ring lines with timestamps; `?follow=1` streams new lines over SSE.
- **IT-010**: HTTP↔UDS parity — `dev`, `reload`, `logs`, `search` return identical payloads through both transports (shared `BaseHandlers` asserted).
- **IT-011**: native tools `compozy__extensions_{init,build,validate,dev,reload,logs,search,provenance,publish}` — all nine invoked through the daemon tool runtime (`publish` against a mock uploader with a server-side credential binding), each returning its structured payload with deterministic errors and no credential value in any output; risk registration asserted: `build`/`dev`/`reload` interaction-gated, `publish` `open_world` + interaction-gated, `validate` read-only.
- **IT-012**: an extension-declared hook fires on its event and `compozy hooks list -o json` reports `source: "extension"` with priority 300.

### SDK conformance + drift

- **IT-013**: public harnesses — `sdk/go/extensiontest` and `@compozy/extension-sdk/testing` each run a conformance fixture per **public** provide capability (tool provider, memory backend, model source, watch source) and fail on a missing required method; bridge-adapter conformance stays in `internal/extensiontest` (in-tree only, ADR-006).
- **IT-014**: end-to-end drift gate — editing an `internal/extension/contract` spec (host methods, hook contracts, or the describe schema) without regenerating fails `make codegen-check` for both TS and Go outputs.
- **IT-015**: cross-workspace isolation — two workspaces on one daemon, each with a dev link of the same extension name: sessions in each workspace see only their own link in list/status/logs/SSE; reload in A never restarts B's generation; each workspace's instance owns its own coordinator, log ring, and last-good generation (`InstanceKey`); operator transports read the global `(name, "")` instance's logs while agent callers in both workspaces are denied them; events carry the correct `workspace_id`.
- **IT-016**: secret-transport absence — with a configured `secret_env` and a publish credential binding, exercise install/dev/reload/logs/publish end to end and assert the raw secret values appear in zero bytes of: HTTP/UDS response bodies, SSE streams, native-tool results, structured errors, persisted events, and the log ring (invariants 13–14).
- **IT-017**: coordinator barrier — concurrent `build` + `reload` + `--watch` cycles against one extension under `-race`: generations advance monotonically, no activation ever reads a partially staged generation, and last-good survives an injected activation failure mid-sequence.

### Contributed commands

- **IT-018**: `GET /api/extensions/commands` over HTTP and UDS returns identical payloads (leaves + groups, workspace-filtered for agent callers); a dev-linked extension's commands appear only for its workspace; every leaf's `flags[]` matches the generated `CommandFlag` contract shape (type/repeatable/required/nullable/enum/default/bounds) with `approval_required` as the approval field.
- **IT-019**: `extension exec cmd-fixture --cmd greet --name X` performs exactly one `POST /api/tools/ext__cmd_fixture__greet/invoke` (transport spy asserts the single call and its input document) and renders the result through the standard bundle in every output format (invariant 16).
- **IT-020**: policy parity — a command whose backing tool requires approval is gated identically through `exec` and through `tool invoke`: unapproved → the same deterministic approval error; approved via `tool approve` → both succeed. A command backed by an unavailable tool reports the tool's reason code.
- **IT-021**: nested execution — `--cmd review/fetch` invokes the leaf's tool id, while `--cmd review` returns the group error without any invoke call reaching the runtime.

## End-to-End Tests

### Authoring journey (R1, R2, R4, R5 — brief Verification: "newcomer path")

- **E2E-001** (runtime lane): on a version-stamped binary and empty home — `compozy extension init hello -t tool-provider-go` → `build` → `validate` → `dev` → tool invoked in a live session returns the fixture answer → edit source → `reload` → changed answer observed → `remove`. Zero trust prompts in the whole loop.
- **E2E-007** (docs-verbatim guard): scripted replay of the published quickstart commands exactly as written — completes with an installed, invocable extension (kills the "cannot be managed-installed as written" class).

### Contributed-command journey (ADR-008 — fixture-proven surface; no product command in this program)

- **E2E-008** (runtime lane): on a stamped binary, install the command-declaring E2E fixture extension (`internal/extension/testdata`-backed; one flat `greet` leaf plus a declared `review` group with a `review/fetch` leaf). `compozy extension commands` renders the tree with the declared group summary; `exec cmd-fixture --cmd greet --name X` performs exactly one tool invocation and renders through `human` and `json`; `--cmd review` errors listing the group's leaves with no invocation reaching the runtime; `--cmd review/fetch` invokes its leaf tool; an approval-gated fixture command is refused until `tool approve`, then succeeds. (The dev-cycle `archive` journey belongs to the `cmd-archive` program.)

### Distribution journey (R2, R10)

- **E2E-002** (runtime lane): `publish` against the mock GitHub server → second isolated home runs `install github:acme/hello` → digest verified → invoke → new release published → `update` picks it up → `remove`.

### Agent journey (R6, R10)

- **E2E-003** (runtime lane): an agent session completes the FULL authoring loop — **scaffold (`extensions_init`) → build (`extensions_build`) → validate → dev → invoke the contributed tool → reload → status → provenance → remove** — using native tools only, every response structured, interaction gates approved in-band; no shell-outs anywhere (peer-review B-004).

### Error UX (R8)

- **E2E-004**: golden-output run of the two most common failures — policy-blocked install and daemon-down — human stderr contains the remediation line and suggested command that `-o json` carries.

### Web surfaces (R3, R4 — Playwright)

- **E2E-005**: marketplace kind page shows a non-zero update count on landing scope, the extension detail renders the Update action, and applying it transitions the card to the new version.
- **E2E-006**: dev-linked extension shows the `dev` badge + `overrides published` label; logs panel streams lines; local-path install form submits through the union contract and surfaces the consent dialog.
