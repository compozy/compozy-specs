---
status: completed
title: "Extension robustness: kit on enable, secrets binding, lifecycle consent & operability (Phases A–C)"
type: backend
complexity: critical
---

# Task 1: Extension robustness — kit on enable, secrets binding, lifecycle consent & operability (Phases A–C)

## Overview

Makes Extension the single kit unit **before** the Bundle product is deleted: everything a bundle could project becomes plain static extension resources live on enable (agents with SOUL/HEARTBEAT sidecars, package automation with enable-as-consent, window layouts, all owner-attributed and convergent), extensions gain the MCP-grade post-install secrets binding path, and the lifecycle gains its serialization authority, digest-exact network consent, and the inventory/preview operability surfaces. This task is TechSpec phases A–C as one vertical slice; task_02 (the cut) depends on every replacement shipped here.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST implement the dir-per-agent static agent contract (`<entry>/<agent>/AGENT.md` + optional `SOUL.md`/`HEARTBEAT.md`/`mcp.json`/`capabilities.toml`), parsing/validating sidecars at register time, rejecting loose `.md` files, and publishing `agent` + `agent.soul` + `agent.heartbeat` records that match in the catalog (TechSpec Core Interfaces; ADR-007).
- MUST add `resources.automation` (TOML `[[jobs]]`/`[[triggers]]`, enabled default true, `event = "webhook"` rejected, targets restricted to same-extension agents) and `resources.layouts` (windowmanager-codec-validated JSON) static loaders with declarative daemon providers; job/trigger names namespaced `<ext>/<name>`, `Source: package`, overlays GC'd when the package definition disappears (ADR-002).
- MUST make ownership executable: `resources.Draft` carries an explicit `Owner`, every extension publisher sets `Owner{Kind:"extension", ID:<name>}` via one shared constructor, managed-record equality compares normalized owner + source, and pre-existing unowned rows converge on the first reconcile. `PackageOwned` accepts the extension owner in this task **alongside** the still-live `bundle.activation` owner; the bundle arm is deleted in task_02 (ADR-004; peer-review B-004).
- MUST return the dedicated `ExtensionEnableResult` action result (namespaced, name-sorted `automation_started` = exactly what the committed operation made runnable) from enable across HTTP/UDS/CLI/native output, never mirrored into status/list payloads, with `extension.enabled` carrying the equal count (peer-review B-005).
- MUST fail enable deterministically (409 `extension_agent_conflict`) when a shipped agent name collides with a visible non-owned agent, surfaced by preview first.
- MUST extend describe/build for code-first kits: SDK definitions declare `resources` dirs, `DescribePayload.resources` is generated into both SDKs, build copies declared dirs into the generation and emits `[resources]`, kit content validates at build/validate with file positions (ADR-008).
- MUST ship migration `00029` (declarative source + Goose + atlas.sum + sqlc): `extension_env_bindings` with `PRIMARY KEY (extension_name, workspace_id, env_name)`, network-confirm columns on `extensions` AND on `extension_dev_links` (peer-review B-001).
- MUST implement the binding path per ADR-003: `vault.ExtensionSecretOwnerPrefix(extension, workspaceID)` dual convention (`global/` + `ws/<workspace>/`), transactional `SetSecrets` (exactly-one-of value/ref per env name, sorted mutation order, reverse rollback, `extension_env`-tagged superseded-ref GC, dangling-ref and undeclared-name rejection), `secrets set|bind|list|unset` verbs + `GET/PUT/DELETE /api/extensions/{name}/secrets[/env]` on both transports with server-side workspace binding.
- MUST resolve spawn env instance-exact: allowlist → manifest `env` → `secret_env` → the launching instance's own bindings, no cross-scope fallback, injection intersected with current `requires_env` (stale bindings listed, never injected); `missing_env` accounts for non-stale bound keys; remove/unlink delete the instance's bindings + GC owned refs; doctor suggests the exact `secrets set` command.
- MUST implement the per-extension-name lifecycle mutation coordinator serializing enable/disable/install/update/remove/dev-link/reload end to end (preflight digest/version fence → staged mutation → confirmation write → reload → reconcile → response; reverse rollback restores registry row, files, confirmation tuple, enabled bit, last-good generation) (peer-review B-002).
- MUST gate network consent digest-exact and actor-true: requests carry `confirm_network_digest`; the coordinator re-reads the candidate manifest and rejects stale digests with the current one; real actor persisted (`operator` / `agent:<id>`); published state on `extensions`, dev state on `extension_dev_links`; update refuses **before any swap** on marketplace-managed sources; `--all` refuses digest-changing items per-item and never blanket-confirms; dev reload honors the same gate (ADR-005; peer-review B-006).
- MUST ship `extension inventory` + `extension preview` as unprivileged GET routes, CLI verbs, and read-only native tools (`compozy__extensions_inventory|preview`), computed from the same desired state as enable (equivalence-tested); `KitItem` identity is `(Kind, Name)` with content-independent shipped IDs (ADR-006).
- MUST emit the canonical events with their required-key matrix (`extension.secrets.updated|update_failed`, `extension.network.confirmed`, `extension.enabled` + count) and extend the coverage-matrix test; SHOULD NOT emit events for consent/validation refusals (deterministic errors own those).
- MUST keep every secret value out of argv, payloads, events, logs, and transcripts (presence-only projections; hidden TTY/stdin intake; redaction registered before injection).
- MUST update `extensions/dev-cycle` fixtures/manifests and in-repo test fixtures to the dir-per-agent contract; MUST regenerate OpenAPI/TS/sqlc/native-tool catalog in the same change (`make codegen-check` green).
- Skills to activate: `eng-code-guidelines`, `golang-master`, `eng-test-conventions`, `testing-boss`, `eng-schema-migration` (00029), `eng-contract-codegen-coship`, `eng-cleanup-failure-paths` (rollback paths), `eng-consolidate-test-suites`.
</requirements>

## Subtasks

- [ ] 1.1 Owner substrate: `Draft.Owner`, shared `extensionOwner` constructor, owner+source-aware managed-record equality, `PackageOwned` extension arm, seeded-row convergence
- [ ] 1.2 Dir-per-agent loader with sidecar parse/validation; agent+soul+heartbeat publication and catalog matching; agent-conflict validation (preview + enable 409)
- [ ] 1.3 Automation static loader (schema, same-extension target restriction, webhook rejection) + declarative provider + scheduler wiring + overlay GC
- [ ] 1.4 Layout static loader + provider (windowmanager codec, global scope)
- [ ] 1.5 `ExtensionEnableResult` contract end to end (core service signature, routes, CLI/native rendering, event count)
- [ ] 1.6 Describe/SDK `resources` declaration + build copy/emit + build-time kit validation (both SDKs, generated contracts)
- [ ] 1.7 Migration `00029` + sqlc (bindings table, confirm columns on `extensions` + `extension_dev_links`) with fresh/reopen/ahead/integrity/equivalence coverage
- [ ] 1.8 Vault ref namer (dual prefix) + binding store + transactional secrets service (rollback, GC, exactly-one-of, sorted order)
- [ ] 1.9 Secrets verbs + routes (both transports, server-side workspace binding, presence-only payloads) + spawn injection (instance-exact, requires_env-intersected, stale semantics) + retirement on remove/unlink + doctor probe
- [ ] 1.10 Lifecycle mutation coordinator wrapping every lifecycle op with fence, commit point, and reverse rollback (including confirmation tuple)
- [ ] 1.11 Network consent gates: enable/update/dev-reload with `confirm_network_digest`, real actor, batch per-item refusal, pre-swap update refusal on marketplace-managed sources
- [ ] 1.12 Inventory + preview services, verbs, routes, native tools; preview↔enable equivalence; `KitItem` identity
- [ ] 1.13 Events + coverage matrix + leak assertions; dev-cycle/fixture updates; `make codegen` regen

## Implementation Details

All interfaces, data-model rationale, invariants (1–17), and endpoint shapes are in `_techspec.md` (Core Interfaces, Data Models, API Endpoints, Safety Invariants); absorb the bundle implementations referenced there before writing replacements — the loaders/preview/confirm semantics are ports of `bundle_agent_loader.go`, `bundle_job.go`, `bundle_layout_loader.go`, `internal/bundles/service_activation.go`, `service_validate_agents.go`, `network_requirement*.go` (do not delete them yet; task_02 owns the deletion). The binding service mirrors `internal/settings/mcp_secrets.go` + `mcp_catalog_install.go`; the automation/layout providers mirror `internal/daemon/loop_resources.go:306-347`.

### Relevant Files

- `internal/extension/{manifest.go,manifest_normalize.go,manifest_load.go,manifest_encode.go,build_describe.go,build.go}` — resource fields, validation, code-first emit
- `internal/extension/manager_resource_loading.go`, `manager_supervision.go`, `manager.go`, `manager_snapshots.go` — register-phase loaders + snapshot
- `internal/extension/{bundle_agent_loader.go,bundle_job.go,bundle_layout_loader.go,bundle_path_security.go}` — semantics being ported (read-only this task)
- `internal/extension/manager_env_resolution.go` (`:29-113`) — spawn env order + redaction seams
- `internal/extension/manifest_network_participation.go` — digest source
- `internal/extension/contract/describe.go` + `internal/codegen/{sdkgo,sdkts}` + `sdk/go`, `sdk/typescript` — describe `resources`, generated contracts
- `internal/daemon/{agent_skill_publisher.go,agent_skill_desired_resources.go,agent_skill_catalog.go,agent_skill_resource_mapping.go}` — sidecar publication + PackageOwned + matching
- `internal/daemon/loop_resources.go` — the declaration-provider pattern to clone for automation/layouts
- `internal/daemon/{extensions.go,extensions_service.go,extension_manager_wiring.go}` — lifecycle service, coordinator home, enable result
- `internal/daemon/native_extension_tools.go`, `internal/tools/builtin_extension_ids.go`, `internal/toolmeta/native_entries.go` — native tools (+2, enable input)
- `internal/vault/{types.go,mcp_refs.go,service.go}` — ref grammar + namer mirror
- `internal/settings/{mcp_secrets.go,mcp_catalog_install.go,mcp_secret_preservation.go}` — the transactional pattern to mirror
- `internal/store/globaldb/schema/definitions/{33_extensions.sql,34_extension_dev_links.sql}` + `schema/migrations/` (next: `00029`) + `queries/` + sqlc
- `internal/automation/{model/types.go,manager_definition_sync.go,manager_runtime_apply.go,manager_webhook_secrets.go}` — package source, overlays, apply
- `internal/windowmanager/{layout.go,resource_codec.go,service_layout_resource.go}` — layout codec/apply
- `internal/api/{contract/extensions.go,core/extensions.go,core/extension_doctor.go,spec/*,httpapi/routes.go,udsapi/routes.go}` — payloads, routes, doctor
- `internal/cli/{extension.go,extension_output.go,extension_state.go}` — verbs + flags + rendering
- `internal/events/extension.go`, `internal/extension/lifecycle_events.go` — event matrix
- `internal/session/soul_start.go`, `internal/soul`, `internal/heartbeat` — sidecar runtime consumption
- `extensions/dev-cycle/**` — conformant kit fixture (already dir-per-agent)

### Dependent Files

- `internal/daemon/agent_skill_resources_integration_test.go` — flat-file agent fixtures must move to dir-per-agent
- `internal/api/{httpapi,udsapi}` handler/parity suites — new routes registered
- `openapi/compozy.json`, `web/src/generated/compozy-openapi.d.ts`, `internal/store/globaldb/sqlcgen/**`, `internal/tools/builtin/testdata/native-tool-catalog.json` — regenerate
- `internal/cli/config_extensions.go` / settings catalog — payload mirrors (`bound_env_keys`, confirm fields)

### Related ADRs

- [ADR-002](adrs/adr-002.md) — enable-as-consent automation; [ADR-003](adrs/adr-003.md) — instance-keyed MCP-shaped binding; [ADR-004](adrs/adr-004.md) — executable extension ownership; [ADR-005](adrs/adr-005.md) — digest-exact lifecycle consent + coordinator; [ADR-006](adrs/adr-006.md) — inventory/preview; [ADR-007](adrs/adr-007.md) — dir-per-agent sidecars; [ADR-008](adrs/adr-008.md) — code-first resources.

### Web/Docs Impact

- `web/`: generated types only this task (`web/src/generated/compozy-openapi.d.ts` via `make codegen`); UI surfaces land in task_03. No hand-written web change.
- `packages/site`: none this task — the docs program is task_04 (site pages, CLI reference regen).
- QA impact: user-visible behavior added (new verbs/routes/flags) — content-addressed `untested` scenarios (`ET-ext-kit-enable`, `ET-ext-secrets-binding`, `ET-ext-inventory`, `ET-ext-preview`, `ET-ext-network-confirm`) are authored in task_04 with the rest of the QA tracker flags; this task records the obligation in its completion notes.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: manifest v2 gains `resources.automation`/`resources.layouts`, agents contract hard-cuts to dir-per-agent; `DescribePayload.resources` + both SDKs regenerate; hook/MCP/bridge surfaces untouched (checked: no new hook events; `resources.mcp_servers` unchanged; bridge slots stay bridge-side).
- Agent manageability: new verbs `inventory|preview|secrets *`, flags `--confirm-network-requirement <digest>`; HTTP/UDS parity for every daemon-backed surface; native tools +2 read-only, enable input extended; deterministic errors per TechSpec (409 confirm w/ digest, 409 agent conflict, 400 binding classes); doctor probe extended.
- Config lifecycle: none — no `config.toml` key added/changed/removed (checked: `[extensions.*]` registry untouched; bindings/confirm are runtime SQLite state by ADR-003 decision).

## Deliverables

- Kit resources (agents+sidecars, automation, layouts) live on enable and removed on disable, owner-attributed and convergent, on a fixture kit and `dev-cycle`
- Secrets binding path end to end (verbs, routes, vault, spawn injection, retirement, doctor)
- Lifecycle coordinator + digest-exact consent gates + `ExtensionEnableResult` + inventory/preview surfaces
- Migration `00029` with full migration-suite coverage; all generated artifacts regenerated
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] UT-001–UT-008 — dir-per-agent loader + sidecar error paths
- [ ] UT-009–UT-014, UT-068 — automation loader (defaults, webhook/duplicate/schedule errors, namespacing, same-extension target restriction)
- [ ] UT-015–UT-017 — layout loader
- [ ] UT-018–UT-021, UT-059 — owner construction, PackageOwned, sidecar matching, owner-aware equality
- [ ] UT-043–UT-046 — describe/build `resources` (round-trip, determinism, position errors, manifest hard cut)
- [ ] UT-047, UT-053, UT-061, UT-070 — enable result rendering + agent-conflict mapping + event count + action-result semantics
- [ ] UT-022–UT-031, UT-065 — ref namer, binding store/service (rollback, GC, overlap, undeclared/dangling), spawn order, stale bindings, missing_env
- [ ] UT-048, UT-054 — secrets list golden + 400 mappings
- [ ] UT-032–UT-037, UT-066, UT-067 — confirm state machine, stale-digest rejection, actor attribution
- [ ] UT-038–UT-042, UT-069 — inventory/preview computation, no-mutation, equivalence, KitItem identity
- [ ] UT-049–UT-052 — inventory/preview goldens, confirm error rendering, jsonl/toon parity, 409 mapping
- [ ] UT-060 — event matrix + leak assertions
- [ ] IT-001 — migration `00029` suites
- [ ] IT-003–IT-006 — kit publish/unpublish/update round trips per kind (+ overlay GC, scheduler start/stop)
- [ ] IT-007, IT-008 — secrets bind→spawn (global + dev instance) + transactional failure
- [ ] IT-009, IT-010, IT-014 — confirm flows (enable, pre-swap update refusal on marketplace-managed sources + batch, dev-link state)
- [ ] IT-013, IT-015, IT-017, IT-018, IT-019, IT-020 — agent conflict, doctor, seeded owner convergence, cross-workspace isolation, lifecycle serialization/rollback, binding retirement
- [ ] E2E-001 — full kit journey (install → secrets → enable+confirm → inventory → invoke → update refusal → confirm → disable)
- [ ] E2E-002 — dev-cycle regression
- [ ] E2E-003 — agent-driven journey through native tools

## Success Criteria

- Every assigned test case implemented and passing
- `make codegen-check`, scoped `make lint` + `go test -race ./internal/extension/... ./internal/daemon/... ./internal/vault/... ./internal/store/globaldb/...` green
- A fixture kit extension proves: enable brings agents(+sidecars)/jobs/triggers/layouts online with owner attribution; disable removes everything including overlays; preview shows the exact publish/conflict/env/automation/digest picture before the act
- No secret value observable in any payload, event, log, or transcript (leak assertions green)
- Bundle surfaces still compile and pass their suites (deletion belongs to task_02)
