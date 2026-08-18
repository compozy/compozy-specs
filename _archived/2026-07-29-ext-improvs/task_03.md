---
status: completed
title: "Code-first toolchain: describe mode, build/validate/init (Phase C)"
type: backend
complexity: high
---

# Task 3: Code-first toolchain — describe mode, build/validate/init (Phase C)

## Overview

Make the SDK definition the single source of truth (ADR-001): both SDKs gain describe mode, `BuildBundle` runs the author's toolchain and emits a deterministic manifest into an immutable generation directory, `ValidateBundle` reports file/line/column issues plus the derived consent summary, and `compozy extension init` scaffolds from embedded templates — deleting the `sdk/create-extension` npm workspace. The dual-declaration/digest-drift failure class becomes structurally impossible (R1/R5).

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST implement `__describe` argv handling in both SDKs printing a valid `DescribePayload` (authoritative schema in `internal/extension/contract`, generated both ways — task_01) and exiting 0.
- MUST implement `BuildBundle` per the TechSpec signature: toolchain detection (`package.json` build script → bun|npm; `go.mod` → `go build`; `BuildCmd` override), describe-mode subprocess with `Timeout` ceiling (default 60s), output into immutable `dist/gen-<content-hash>/`, `BuildResult.GenerationHash`, closed-set validation before any manifest write; `<SourceDir>/dist` is the only dev-linkable root.
- MUST make manifest generation deterministic (sorted keys, stable ordering — invariant 11) and stamp `min_compozy_version` from `DescribeSDKInfo` (hand-authored values ignored); describe mode runs only in author-owned flows (invariant 12).
- MUST implement `ValidateBundle` returning `ValidationIssue{Path,Line,Column,Field,Message,Severity}` with the closed `IssueSeverity` enum and the derived consent-area summary.
- MUST add daemon-free CLI verbs `init` (embedded FS templates: tool-provider TS, go-tool-provider, hook TS, memory-backend TS, loop-watch-source Go — ported from `sdk/create-extension/templates/`, corrected: no phantom capabilities, no hand-authored `min_compozy_version`), `build`, `validate`.
- MUST delete `sdk/create-extension/` entirely (delete target) and add the CI gate keying in-repo manifests to the stamped daemon version (the 0.5.0-vs-0.3 class dies).
- MUST keep retained reconciliation (`ReconcileManifestToolRuntime`) as an install/activate defense-in-depth assert only.
</requirements>

## Subtasks

- [x] 3.1 Describe mode in `sdk/go` and `sdk/typescript` (argv handling + payload assembly from registrations)
- [x] 3.2 `internal/extension/build.go`: `BuildBundle` (detect → build → describe → validate → emit generation) + determinism + hash
- [x] 3.3 `ValidateBundle` with positioned issues + consent summary
- [x] 3.4 `compozy extension init|build|validate` verbs (daemon-free) + embedded templates (five, corrected content)
- [x] 3.5 Delete `sdk/create-extension/`; CI manifest-version gate
- [x] 3.6 Implement every assigned test case (build determinism is a tested invariant)

## Implementation Details

TechSpec: Core Interfaces (BuildRequest/BuildResult/ValidateBundle/IssueSeverity), Component Overview data flow, Delete Targets, Safety Invariants 11–12. Phase C gate: build-determinism test + e2e build→install fixture.

### Relevant Files

- `internal/extension/{describe.go,tool_reconciliation.go}` — existing describe consumption + retained reconciliation assert
- `internal/extension/build.go` — new component (suites: new `build_test.go`, `build_integration_test.go`)
- `sdk/go/extension.go` + `sdk/typescript/src/extension-runtime.ts` — describe-mode entry points
- `internal/cli/extension.go` — verb registration (init/build/validate); embedded templates under `internal/cli`
- `sdk/create-extension/{src/index.ts,templates}/` — source of the five templates (`scaffoldExtension` L90, `TEMPLATE_NAMES` L24); whole workspace deleted after porting. Known template rot the port fixes by construction: `private: true` package, manifest-format inconsistency (2 templates ship `extension.toml`, 3 ship `extension.json`) — generated manifests make it moot
- `internal/extension/manifest_compatibility.go` — version-gate behavior the CI gate keys on

### Dependent Files

- `internal/daemon/native_extension_tools.go` — `extensions_init`/`extensions_build`/`extensions_validate` native tools reuse these daemon-free services (registered in task_04/05 waves per Agent Manageability)
- `bun.lock`, root `package.json` — workspace removal side effects

### Related ADRs

- [ADR-001: Code-first authoring](adrs/adr-001.md) — primary
- [ADR-003: Uniform SDK architecture](adrs/adr-003.md) — describe schema generation (task_01 dependency)

### Competitor References

- `.resources/eve/packages/eve/src/setup/scaffold/create/extension.ts:48-180` — manifest generated from the definition at build time (the ADR-001 precedent)
- `.resources/zed/crates/extension_cli/src/main.rs:27-90` — packaging CLI validating before publish
- `.resources/eve/docs/extensions.md` — the authoring-journey bar (concepts/actions counts)

## Web/Docs Impact

none in this task — quickstart/`develop.mdx` rewrite around init→build→validate is owned by task_09; no `web/` change.

**QA impact**: add content-addressed `untested` scenarios for the new authoring entry (`extension init` → `build` → `validate` first success); no existing scenario resets (verbs are new).

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: the manifest becomes a build artifact; resource-only extensions keep minimal handwritten manifests (validated only). Templates are the authoring on-ramp for all four public provides.
- Agent Manageability: `init|build|validate` are daemon-free CLI verbs with structured output (`issues[]`, consent areas); their native-tool exposure (same services, identical outputs) lands with the daemon waves (task_04/05).
- Config Lifecycle: none — checked surfaces: no new `config.toml` key (`extensions.dev.watch_interval` is task_04's consumer; defined in task_02).

## Deliverables

- One-declaration authoring live: code → `build` → generated manifest, byte-deterministic
- `init` scaffolds a working extension with zero phantom concepts
- `sdk/create-extension` gone; CI version gate green
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] UT-008, UT-009, UT-010, UT-011, UT-012, UT-013, UT-014 — toolchain detection, determinism, timeout, unknown provide, stamping, positioned issues, consent payload
- [x] UT-041 — Go `Tool[T]` registration reaches the describe payload with schema digests
- [x] UT-046 — TS `__describe` prints a valid payload (TestHarness)
- [x] IT-005 — describe-mode round trip: real Go fixture subprocess → build → install → tool visible in the registry

## Success Criteria

- Every assigned test case implemented and passing
- Two builds over identical source produce byte-identical `extension.toml` (invariant 11 test green under `-race`)
- `compozy extension init hello -t tool-provider-go && compozy extension build && compozy extension validate` succeeds on a clean machine with only the binary
- Brief scorecard trajectory: the authoring path counts ≤4 actions to first success (measured in task_09/QA)

## Completion Notes

- Contract parity: `_techspec.md` Core Interfaces, Safety Invariants 11–12, ADR-001, and `_tests.md` UT-008–014/041/046 plus IT-005 are implemented. IT-005 builds a real Go SDK subprocess, installs and activates it, resolves the generated tool through the central registry, and invokes it successfully.
- Fresh focused verification: `make lint`; `go test -race ./internal/extension/... -count=1` (850); `go test -race ./internal/cli/... -count=1` (1,334); tools/codegen/OpenAPI race suites (857); Go SDK race suite (68); TypeScript SDK Turbo lane (54 plus typecheck and codegen); IT-005 (1); Windows cross-compile; and a real binary `init → build → validate` smoke all pass. The program-wide `make verify` remains reserved for the final 11-task gate by the loop cadence.
- QA impact: new user-visible CLI behavior is flagged by `ET-extension-code-first-authoring` with `qa_status: untested`; tracker materialization validates 659 scenarios.

Compozy Impact Audit:

- Native tools: no tool IDs, toolsets, descriptors, schema digests, risk flags, availability diagnostics, or capability gates changed; checked `internal/daemon/native_extension_tools.go`. Task 03 exposes reusable daemon-free services, while `extensions_init|build|validate` native descriptors are owned by Tasks 04–05.
- Extensibility and hooks: material impact — both SDKs now emit the authoritative describe contract; builds generate immutable manifest v2 bundles for tools, hooks, memory backends, and watch sources; five embedded templates cover the public provides; reconciliation remains an install/activate assertion. No config lifecycle change after checking the Task 02 `extensions.trust|sources|dev` surface.
- Workspace data isolation: no persisted runtime datum — `init|build|validate` operate on explicit local paths without daemon, store, cache, SSE, event, HTTP, or UDS access. Therefore no global/workspace/session/agent ownership or `workspace_id` propagation changes occur in this task; trusted workspace binding starts with the dev/native surfaces in Task 04.
- Official Compozy skill: no file changed yet after checking `skills/compozy/`; the new public authoring flow is intentionally documented there together with site docs in Task 09, after the Task 04–08 surfaces settle.
