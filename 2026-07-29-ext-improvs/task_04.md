---
status: completed
title: "Dev lane: links, instance-keyed reload, logs, watch (Phase D)"
type: backend
complexity: critical
---

# Task 4: Dev lane — links, instance-keyed reload, logs, watch (Phase D)

## Overview

Ship the author's inner loop (R2/R4): `compozy extension dev` links a built generation with zero trust ceremony, `reload` swaps generations atomically under a per-instance coordinator, `logs` streams a redacted per-instance ring, and `--watch` closes edit→reload→observe. This is the program's safety-primitive task: workspace identity as an authorization and concurrency boundary (`InstanceKey`), content-hash generation handles, last-good recovery — safety invariants 1–3, 7–8, 12(dev), 14–15 live here.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST implement `Registry.LinkDev/UnlinkDev/ResolveActive` over the `extension_dev_links` side table (task_02): upsert by `(extension_name, workspace_id)` with no 409, published row never displaced, unlink = side-table delete (invariant 1: only `dev` creates links; `install` can never mint one).
- MUST introduce `InstanceKey{Name, WorkspaceID}` keying every runtime dev surface — subprocess, operation coordinator, last-good generation, log ring, status, events; global published installation = `(name, "")`; agent callers reach only instances their workspace resolves to; global-instance logs are operator-transport-only (invariant 2).
- MUST bind `workspace_id` server-side only (operator: resolved workspace; agent: trusted session scope via the canonical dispatcher) — never from request bodies or tool inputs; agent-facing list/status/logs/SSE/event projections filter by it (L-033).
- MUST canonicalize origin paths (`realpathDeepestExisting` + `EvalSymlinks` containment under the workspace root, macOS `/private/var` quirk handled) at link time and every load; missing/escaping origin → `errored (missing_origin)`, never a boot crash (invariant 3).
- MUST make generation identity a content-hash handle only: `DevLinkRequest{origin_path, generation_hash}` / `ReloadExtensionRequest{generation_hash}`; the daemon reconstructs `<origin>/dist/gen-<hash>`, validates hash format, and re-verifies manifest + digest before activation; `ErrExtensionGenerationInvalid` 400, `ErrExtensionNotDevLinked` 409 (invariant 3, R2 B-002).
- MUST serialize build/stage/swap/reload/watch through one per-instance operation coordinator; failed activation restarts the last-good generation with status `errored (activation_failed; running <prior generation>)` (invariants 7–8).
- MUST feed a per-instance bounded ring buffer (256 KiB, drop-oldest, never blocks) from subprocess stderr with secret redaction at ingestion, upstream of every transport (invariants 14–15).
- MUST ship the full surface in one pass (no partial surface): `POST /api/extensions/dev`, `POST /api/extensions/{name}/reload`, `GET /api/extensions/{name}/logs` (`?follow=1` SSE via named event `extension_log` per L-017) with UDS parity via `internal/api/core` shared handlers; CLI `dev|reload|logs` (+ `--watch` CLI-side poll via `internal/filesnap`, cadence `extensions.dev.watch_interval`); native tools `compozy__extensions_{init,build,validate,dev,reload,logs}` (init/build/validate reuse task_03 services; dev/reload interaction-gated, validate read-only); contract payloads + OpenAPI + generated TS + E2E mocks co-ship (L-007).
- MUST detach nothing to request lifetime (SD-010) and track every goroutine (ring feeder, watcher) with owned shutdown joins; Windows cross-build gate applies (no symlinks — link is a registry row).
</requirements>

## Subtasks

- [x] 4.1 `internal/extension/dev.go` (link records + resolution) + `instance.go` (`InstanceKey` + resolution rule)
- [x] 4.2 `internal/extension/manager_lifecycle.go`: per-instance coordinator, generation re-verification, atomic swap, last-good restart
- [x] 4.3 Origin canonicalization + generation-handle validation (hostile-input surface)
- [x] 4.4 `internal/extension/logs.go`: per-instance ring + ingestion-time redaction
- [x] 4.5 Routes + UDS parity + server-side workspace binding + new `ExtensionStatusCode` entries
- [x] 4.6 CLI `dev|reload|logs` + `--watch` (filesnap poll, debounce, serialized reloads)
- [x] 4.7 Native tools registration (six through this task) with risk classes; composition-root wiring only
- [x] 4.8 Contract payload additions (`overrides_published`, `origin_path`, dev label projection) + `make codegen` + E2E mocks
- [x] 4.9 Implement every assigned test case (concurrency under `-race`; hostile matrices; E2E-001 journey)

## Implementation Details

TechSpec: Core Interfaces (dev lane block), Data Models (`DevLinkRequest`), API Endpoints (instance semantics paragraph), Safety Invariants 1–3/7–8/14–15, Agent Manageability (native tools + deterministic errors). Phase D gate: e2e dev loop.

### Relevant Files

- `internal/extension/{registry.go,registry_types.go,manager.go,manager_failure_state.go}` — resolution + lifecycle integration points (manager files are large: extract new files, never grow past the cap)
- `internal/api/core/extensions.go` — shared handlers (suite: `extensions_test.go` owns parity + status codes)
- `internal/api/httpapi` + `internal/api/udsapi` — route registration (SSE suite: new `extension_logs_sse_test.go`)
- `internal/daemon/{extensions.go,extension_manager_wiring.go,native_extension_tools.go}` — composition-root wiring (SD-008)
- `internal/cli/{extension.go,extension_state.go}` + `internal/filesnap/` — verbs (registration block `extension.go:55-63`) + watch loop
- `internal/config` — `extensions.dev.watch_interval` consumer (key defined in task_02)
- `internal/testutil/e2e/{runtime_harness.go,bridges_extensions.go}` — `StartRuntimeHarness` + extension helpers the E2E-001 journey builds on (new journey file follows the `//go:build integration && !windows` + `TestDaemonE2E<Behavior>` pattern)

### Dependent Files

- `web/src/systems/extensions/*` — dev badge/logs panel consume these payloads (task_08)
- `internal/observe` event consumers — dev/reload events with per-event keys (matrix completed in task_07)

### Related ADRs

- [ADR-002: First-class dev lane](adrs/adr-002.md) — primary
- [ADR-001: Code-first authoring](adrs/adr-001.md) — dev builds via `BuildBundle`
- [ADR-007: Extension config consolidation](adrs/adr-007.md) — watch cadence key

### Competitor References

- `.resources/zed/crates/extension_host/src/extension_host.rs:1032-1133` — dev-install/rebuild two-lane precedent (last-good behavior)
- `.resources/claude-code/commands/reload-plugins/reload-plugins.ts` + `.resources/claude-code/cli/handlers/plugins.ts:175-177` — reload verb + `--plugin-dir` dev lane
- `.resources/pi/packages/coding-agent/docs/extensions.md` — live `/reload` inner-loop bar
- `.resources/openclaw/docs/tools/plugin.md:373` — local dev install (`plugins install -l`)

## Web/Docs Impact

`web/` consumes this task's payload additions in task_08 (dev badge, `overrides published`, logs panel via `GET /api/extensions/{name}/logs` SSE); generated types refresh here via `make codegen` with E2E mocks co-shipped (L-007). Site docs (dev-loop guide) owned by task_09.

**QA impact**: add content-addressed `untested` scenarios for the dev inner loop (dev → invoke → edit → reload → observe; logs follow; unlink restores published); reset any ET-015..ET-023 scenario that exercises reinstall-as-iteration (the flow it replaces).

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: the dev overlay is a new lifecycle surface for every subprocess extension; no manifest schema change beyond task_02/03.
- Agent Manageability: six native tools (init/build/validate/dev/reload/logs) with structured outputs and deterministic errors; HTTP/UDS parity; workspace-scoped agent projections. Interaction gates: build/dev/reload approval-gated, validate read-only.
- Config Lifecycle: consumes `extensions.dev.watch_interval` (defined task_02); no new keys — checked: `internal/config` diff limited to consumption.

## Deliverables

- Zero-ceremony dev lane: one command links, reload swaps atomically, watch loops, logs stream redacted
- Instance-keyed isolation proven cross-workspace; global instance operator-only logs
- Full surface parity (routes/UDS/CLI/native tools/payloads/codegen) in one pass
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] UT-015, UT-016, UT-017, UT-018 — link upsert/resolution/published-row integrity/install-can-never-mint (invariant 1)
- [x] UT-019, UT-020, UT-021 — missing origin, per-instance coordinator race (InstanceKey + hash), last-good restart
- [x] UT-062, UT-067 — ring bounds under concurrency; ingestion-time redaction across ring/Logs/SSE
- [x] UT-064, UT-065, UT-066 — hostile workspace binding (incl. global-instance operator/agent rule), symlink escape, operator binding
- [x] UT-086 — generation-handle hostile matrix (`ErrExtensionGenerationInvalid`)
- [x] IT-006 — full dev loop with hash re-verification
- [x] IT-009 — logs route + SSE follow (named event)
- [x] IT-015 — cross-workspace isolation: per-instance coordinators/rings/last-good; operator-vs-agent global logs; event `workspace_id`
- [x] IT-017 — coordinator barrier under `-race` with injected activation failure
- [x] E2E-001 — authoring journey on a stamped binary: init → build → validate → dev → invoke → edit → reload → observe → remove, zero trust prompts

Contract co-ship gates (L-007): `make codegen && make codegen-check` after contract edits + `bunx turbo run typecheck test --filter=./web` (regenerated types compile; E2E mocks updated in the same change).

## Success Criteria

- Every assigned test case implemented and passing (`-race` clean)
- A forged `workspace_id` or hostile `generation_hash` can never read, mutate, or execute across the boundary (hostile matrices green)
- Broken edit never takes the extension down: last-good keeps serving with honest status
- `GOOS=windows GOARCH=amd64 go build ./...` passes

## Completion Notes

Implemented the complete Phase D author loop across registry, runtime, transports, CLI, native tools,
events, generated contracts, official skill guidance, and the generated-contract Web consumers. Dev
instances are keyed by `(name, workspace_id)` and retain their own coordinator, process, last-good
generation, log ring, status, and events. Generation handles are re-verified content hashes, reloads
swap atomically, failed activation restores the verified last-good runtime, and stderr is bounded and
redacted before any consumer can observe it.

Fresh focused evidence after source freeze:

- `make lint` — PASS, zero issues; source-size and source-policy gates PASS.
- `go test -race ./internal/extension` — 835 tests PASS.
- `go test -race -tags=integration ./internal/extension -run 'TestManagerDevelopment|TestVerifyDevGeneration' -count=1` — 12 tests PASS.
- `go test -race ./internal/api/core ./internal/api/httpapi ./internal/api/udsapi ./internal/cli ./internal/daemon ./internal/subprocess ./internal/tools ./internal/tools/builtin` — PASS.
- `go test -race -tags=integration ./internal/daemon -run 'Test(NativeExtensionToolsIntegrationLifecycleParity|DaemonE2EExtensionAuthoringShouldCompleteTheDevelopmentLoopWithoutTrustPrompts)$' -count=1` — PASS.
- `bunx turbo run lint typecheck test --filter=./web` — 5/5 tasks, 515 files and 4,046 tests PASS.
- `npx react-doctor@latest --verbose --diff` — 100/100.
- `make codegen-check` and `GOOS=windows GOARCH=amd64 go build ./...` — PASS.
- Storybook captures inspected at `/tmp/eng-ui-screenshot.QgUi28/evidence/` for Settings/Extensions and the installed extension detail; the owned Storybook process was torn down.
- QA tracker materialization — 660 scenarios; new `ET-extension-dev-reload-loop` remains `untested` by design.

The single program-wide `make verify` remains deferred until all eleven tasks are complete, per the
accepted loop execution contract.

Compozy Impact Audit:

- Native tools: added `compozy__extensions_{init,build,validate,dev,reload,logs}` descriptors, input schemas, risk/interaction flags, availability wiring, catalog golden coverage, and lifecycle integration. `validate`/`logs` are read-only; `build`/`dev`/`reload` require interaction.
- Extensibility and hooks: added the workspace dev-overlay registry/runtime, immutable-generation activation, extension tool/resource authority, lifecycle events, CLI watch consumption of `extensions.dev.watch_interval`, and official `skills/compozy/` guidance. No new manifest or config key was introduced; Task 02/03 contracts remain authoritative.
- Workspace data isolation: dev links, processes, coordinators, last-good state, rings, status, SSE/log reads, and events are workspace-scoped. Operator workspace resolution and trusted agent session scope bind server-side across CLI/HTTP/UDS/core/native dispatch; agent global-log access and cross-workspace reads are denied by tests.
- Official Compozy skill: updated `skills/compozy/SKILL.md`, `references/capabilities-and-bundles.md`, and `references/native-tools.md` with the exact authoring/dev loop, deterministic errors, trust model, scopes, hashes, last-good behavior, logs, and watch lifecycle.

Web/Docs Impact:

- `web/`: regenerated OpenAPI types and hard-cut stale `extensions.marketplace`/`actions` consumers to `sources`, `trust`, and `permissions`; updated fixtures, Storybook, and E2E selectors/config seeding. Task 08 still owns the new dev badge/log panel product UI.
- `packages/site`: no runtime guide changed; checked the Task 09 ownership boundary, which owns the public dev-loop guide after all extension surfaces settle.
- QA: added content-addressed `docs/qa/scenarios/ET-extension-dev-reload-loop.md`; ET-015..ET-023 were already `untested`, so no stale verdict required resetting.
- Tooling-only cross-build repair: `scripts/air-state-lock` and its process-integration suite are now explicitly Unix-only; no user-visible behavior changed.
