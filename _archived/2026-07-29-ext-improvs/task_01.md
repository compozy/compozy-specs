---
status: completed
title: Generated SDK contracts + publishing groundwork (Phase A)
type: backend
complexity: high
---

# Task 1: Generated SDK contracts + publishing groundwork (Phase A)

## Overview

Build the single generated-contract foundation every later task consumes: a Go contract generator mirroring the TS one, both fed from `internal/extension/contract`, so the daemon, Go SDK, and TS SDK can never drift (R5–R7). Restructure `sdk/go` into an installable nested module, make the npm package public, wire both into the release pipeline, and ship the public conformance harness — the adoption wall (unobtainable SDKs, hand-maintained wrong maps) falls here.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST add `internal/codegen/sdkgo` mirroring the `sdkts` structure, reading only `internal/extension/contract` sources (`HostAPIMethodSpecs()`, `BuildHookContracts()`, `SDKRootTypes()`, describe schema), and a `sdk-contracts-go` target in `cmd/compozy-codegen` wired into `make codegen`/`codegen-check` (byte-checked, invariant 10).
- MUST emit into `sdk/go/contracts/`: all 87 host-method constants with typed params/results, all 90 hook events with typed payloads/patches, the describe-payload schema, conformance fixtures, and `RequiredMethods(provide)`.
- MUST single-source the capability→required-methods map: the generated map replaces the hand maps in BOTH SDKs (delete `sdk/go/types.go:406-411` and `REQUIRED_PROVIDES_METHODS` in `sdk/typescript/src/capabilities.ts:8-13`); `bridge.adapter` gains `bridges/targets/snapshot`, `model.source` gains `models/list`.
- MUST propagate `trusted_workspace` and `invocation_id` daemon → handler verbatim in both SDKs (invariant 9): `ExtensionToolCallRequest` mirrors gain `TrustedWorkspace`/`InvocationID`; TS `parseToolCallRequest` stops dropping them.
- MUST make `sdk/go` a nested module `github.com/compozy/compozy/sdk/go` (own `go.mod`, internal `replace` for in-repo consumers, release tag `sdk/go/vX.Y.Z`) importing nothing under `internal/*` (extend `TestSDKHasNoDaemonInternalImports` to the module boundary + boundaries rule in the same commit).
- MUST flip `@compozy/extension-sdk` to public npm posture (real license) and wire npm publish + sdk/go tagging into `.github/workflows/release.yml` (`release` job; validated via the `dry-run` job).
- MUST ship the public conformance harness `sdk/go/extensiontest` covering exactly the four public provides (`tool.provider`, `memory.backend`, `model.source`, `loop.watch_source`); bridge-adapter conformance stays `internal/extensiontest` (ADR-006).
- MUST register `internal/codegen/sdkgo` in `magefiles/boundaries.go` in the same commit that introduces it (SD-008).
</requirements>

## Subtasks

- [x] 1.1 `internal/codegen/sdkgo` generator package (collect specs → emit Go files) + `cmd/compozy-codegen` `sdk-contracts-go` target with check mode
- [x] 1.2 Generated `sdk/go/contracts/` layer: host methods, hook events, describe schema, `RequiredMethods`, conformance fixtures
- [x] 1.3 Replace both hand-written required-method maps with generated output (delete targets); fix TS `capabilities.ts` consumption
- [x] 1.4 `TrustedWorkspace` + `InvocationID` parity in `sdk/go` and `sdk/typescript` request mirrors and handler surfaces
- [x] 1.5 `sdk/go` nested module: `go.mod`, internal `replace`, module-boundary test, boundaries rule
- [x] 1.6 npm public posture + `release.yml` publish/tag wiring, exercised by the release `dry-run` job
- [x] 1.7 Public harness `sdk/go/extensiontest` (four provides) alongside the existing TS `@compozy/extension-sdk/testing`
- [x] 1.8 Implement every assigned test case; run `make codegen-check` + scoped `go test -race` + `bunx turbo run test --filter=./sdk/typescript`

## Implementation Details

TechSpec: Component Overview (`internal/codegen/sdkgo`, `sdk/go`, `sdk/typescript`), Core Interfaces (generated contracts excerpt), Architectural Boundaries, Impact Analysis rows for `sdk/*` and `release.yml`. Phase A gate: `make codegen-check`, scoped `go test`, `bun-test`, boundaries.

### Relevant Files

- `internal/codegen/sdkts/generate.go` — structure to mirror (imports `internal/extension/contract` aliased `extensioncontract`)
- `internal/extension/contract/{host_api_method_registry.go,sdk.go,sdk_contract_builder.go,sdk_named_types.go}` — spec sources the generator reads
- `cmd/compozy-codegen/main.go` — target registration + check mode (existing `main_test.go` owns UT-040/070/IT-014)
- `sdk/go/{types.go,extension.go,tool_request_test.go,runtime_contract_test.go}` — hand map delete target; request mirror; boundary test to extend
- `sdk/typescript/src/{capabilities.ts,extension-runtime.ts,generated/contracts.ts,testing/}` — hand map delete; parse fix; harness sibling
- `.github/workflows/release.yml` — `release` job (L409) publish steps; npm auth precedent at L149–155; channel-policy verification (`npm view … dist-tags`, L711–720) gains the SDK package; the annotated-tag step gains the `sdk/go/vX.Y.Z` tag; `dry-run` job (L114) validates
- `.goreleaser.yml` — existing `npms:` block (L88–100, `@compozy/cli`) — a second `npms:` entry is the lowest-friction SDK publish path
- `cmd/compozy-codegen/main.go` — targets switch (L57–86): new `sdk-contracts-go` = path const + `case` + write/check pair beside `defaultSDKContractsPath` (L26)
- `sdk/typescript/package.json` + `sdk/go` module absence — both publish blockers (`private: true`; no `go.mod`) die here
- `magefiles/boundaries.go` — new-package registration

### Dependent Files

- `internal/extension/tool_provider.go`, `internal/tools/provider_descriptor.go:54` — the daemon side already sends `trusted_workspace`; SDK mirrors must match its wire shape
- `sdk/examples/*` — will consume the public module (cleaned in task_09)

### Related ADRs

- [ADR-003: Uniform SDK architecture](adrs/adr-003.md) — the three-layer design and generated-contract authority this task implements
- [ADR-006: Closed-surface positioning](adrs/adr-006.md) — `bridge.adapter` excluded from the public harness/completeness surface
- [ADR-001: Code-first authoring](adrs/adr-001.md) — describe schema is part of the generated contract set (consumed in task_03)

### Competitor References

- `.resources/pi/packages/coding-agent/src/core/extensions/types.ts:1185-1412` — the per-event typing bar the generated hook contracts must meet
- `.resources/eve/scripts/extension-capability-contracts.mjs:38-74` — machine-verified capability contracts (the drift-gate precedent)

## Web/Docs Impact

none — backend/SDK groundwork; no `web/` route or component changes. Site docs (SDK install instructions, reference pages) are owned by task_09.

**QA impact**: none — no user-visible runtime behavior change (publishing wiring is exercised by the release `dry-run` job, not a QA scenario).

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: generated contracts become the single authority feeding both bridge-SDK groundwork (bridge service methods emitted as internal groundwork per ADR-006) and every later surface; no manifest/hook/tool schema change in this task.
- Agent Manageability: no CLI/HTTP/UDS change; `make codegen-check` is the deterministic drift gate agents rely on.
- Config Lifecycle: none — checked surfaces: `internal/config` untouched; no `config.toml` key added/removed.

## Deliverables

- `internal/codegen/sdkgo` + generated `sdk/go/contracts/` passing `make codegen-check` in both languages
- Both hand maps deleted; parity fields reaching handlers in both SDKs
- Installable `sdk/go` module + public npm package + release wiring validated by dry-run
- Public `sdk/go/extensiontest` harness
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] UT-038, UT-039, UT-040 — generator counts, generated `RequiredMethods` correctness, check-mode staleness
- [x] UT-042, UT-043, UT-044, UT-045 — Go SDK: trusted_workspace verbatim, clarify `invocation_id`, `capability_denied`, module boundary
- [x] UT-047, UT-048, UT-049 — TS SDK: trustedWorkspace/invocationId, clarify, generated-map initialize validation
- [x] UT-070 — describe-schema drift fails `codegen-check` for both SDKs
- [x] IT-013 — public harnesses run a conformance fixture per public provide (bridge stays internal)
- [x] IT-014 — end-to-end drift gate on `internal/extension/contract` edits

## Completion Notes

QA tracker impact: no user-visible UI, CLI verb, API route, config key, or public copy changed; release packaging is covered by the package dry-run and does not add a runtime QA scenario.

Compozy Impact Audit:

- Native tools: no impact — checked `internal/tools` native IDs, descriptors, schemas, digests, risk flags, and capability gates; generated SDK contracts consume the existing host/tool authority without changing any `compozy__*` surface.
- Extensibility and hooks: generated Go and TypeScript contracts now cover the authoritative Host API, hook event, describe, and provide-method registries; public conformance exposes `tool.provider`, `memory.backend`, `model.source`, and `loop.watch_source`, while `bridge.adapter` remains in-tree only. No extension manifest, bundle, MCP sidecar, or `config.toml` lifecycle key changes in this task.
- Workspace data isolation: `trusted_workspace` and `invocation_id` remain invocation-scoped and now propagate verbatim through both SDK handlers; no global/workspace/session/agent storage, list, cache, SSE, or event ownership path changed. Go and TypeScript handler tests cover the scope propagation.
- Official Compozy skill: no impact — checked `skills/compozy/`; it does not currently document SDK package installation or generated contract APIs, and the public documentation update is owned by task 09.

## Success Criteria

- Every assigned test case implemented and passing
- `make codegen-check` green with zero handwritten edits to generated files; mutating any generated byte fails CI
- `go get github.com/compozy/compozy/sdk/go` resolves standalone (verified in a CI job consuming the SDK as an external module)
- Release `dry-run` job exercises npm publish + module tagging without error
