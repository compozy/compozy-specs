---
status: completed
title: Rebrand wire protocol and protocol documents
type: refactor
complexity: high
---

# Task 03: Rebrand wire protocol and protocol documents

## Overview

Third rebrand class: the network wire identity family (`agh-network/*` → `compozy-network/*`), the runtime peer id, envelope extension and digest prefixes, direct-room identity derivation, the raw claim-token identity family, and loop `apiVersion` (`agh.loop/v1` → `compozy.loop/v1`). This class is the highest co-ship risk in the program: persisted identifiers, security redaction, RFC filenames, web loop rendering, site protocol docs, and pinned tests must move in one hard cut.

<critical>
- ALWAYS READ `_brief.md`, `_techspec.md`, `_content-plan.md`, `_tests.md`, every ADR, and any per-task memory before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — persisted values are invalidated by the clean restart; do not add an old-wire reader, alias, or fallback
</critical>

<requirements>
- MUST rename the protocol constant `ProtocolV0` value to `compozy-network/v0` at its single definition site.
- MUST rename `RuntimePeerID` and the derived runtime peer session id to the `compozy.runtime` form.
- MUST rename all eight `agh.*` envelope extension keys (capability brief, include, capability ids, capability catalog, workflow id, handoff version/digest/source) at both write and read sites.
- MUST hard-cut the complete raw claim-token family from `agh_claim_` / case-insensitive `AGH_CLAIM_*` fixtures to `compozy_claim_` / `COMPOZY_CLAIM_*` across generation, redaction, JSON safety, stores, transports, diagnostics, fixtures, QA guidance, and public security documentation. The public field names remain `claim_token` and `claim_token_hash`; raw tokens remain forbidden from every response, log, event, transcript, Web surface, and persisted diagnostic.
- MUST rename the live `agh.soul.*` and `agh.heartbeat.*` digest/wake identifiers, accepting persisted-digest invalidation without a compatibility reader.
- MUST rename the direct-room derivation prefix `agh-network/direct-room/v1` to `compozy-network/direct-room/v1`, update its canonical vector, and record that existing direct-room IDs require a clean restart.
- MUST rename the loop `apiVersion` to `compozy.loop/v1` at the DSL declaration, the public DTO, every embedded loop YAML, and every matching test fixture.
- MUST co-ship test matchers and persisted fixtures in the same commit as the wire rename (TechSpec Safety Invariant 10) — no commit may leave a matcher asserting a renamed string.
- MUST rename `docs/rfcs/003_agh-network-v0.md` and `docs/rfcs/004_agh-network-v1.md` to their `compozy-network` filenames and sweep every matching protocol RFC body (003–005 today).
- MUST update `packages/site/lib/__tests__/protocol-rfc-hard-cut.test.ts` in the same change — it asserts the PRESENCE of the old wire string and pins RFC filenames at the repo root.
- MUST sweep `packages/site/content/protocol/`, `packages/site/content/runtime/core/network/*`, and every active protocol-family/profile literal, including future-profile identifiers.
- MUST update active web loop DSL text, fixtures, tests, and any shared-UI story that renders `agh.loop/v1`; this is a value-contract co-ship, not a later branding-only sweep.
- MUST update the existing official `skills/agh/` network and loop references for `compozy-network/v0`, `compozy.loop/v1`, peer IDs, and public protocol values in this same hard cut. Task 04 owns the directory/name/metadata rename, not stale protocol guidance.
- MUST NOT invent a NATS subject-prefix constant. Branded digest prefixes and direct-room derivation do exist and are in scope.
- MUST accept persisted-digest invalidation as in-policy (clean restart per brief) and MUST NOT add a compatibility reader for old envelope keys.
</requirements>

## Subtasks

- [x] 3.1 Rename the protocol version constant and runtime peer identity
- [x] 3.2 Rename all envelope, digest, and direct-room prefixes at write and read sites, including greet-summary, API, CLI output, and canonical vectors
- [x] 3.3 Rename the loop `apiVersion` across DSL, DTO comments, embedded loop YAMLs, active web rendering, shared-UI stories, and test fixtures
- [x] 3.4 Update network transport/envelope, HTTP/UDS/CLI, web, and persisted-fixture matchers in the same change
- [x] 3.5 Rename the two RFC files and sweep every matching protocol RFC body (003–005 today)
- [x] 3.6 Sweep site protocol content, runtime network docs, direct-room material, and future-profile identifiers
- [x] 3.7 Update `protocol-rfc-hard-cut.test.ts` (presence assertion + pinned RFC filenames) and the glossary entries it reads
- [x] 3.8 Update the official `skills/agh/` network/loop protocol guidance in the same hard cut
- [x] 3.9 Hard-cut the complete raw claim-token family while preserving field names and cross-transport leak protection
- [x] 3.10 Implement UT-059
- [x] 3.11 Run the wire-class grep gate to zero; run `make verify` and the site test lane

## Implementation Details

Single-commit cross-tree co-ship: Go constants and persisted prefixes, raw claim generation plus every redaction/leak-defense consumer, RFC filenames at the repo root, web loop value rendering, site MDX, and the site spec that pins both. Follow TechSpec §Delete Targets (identity hard cut), §Safety Invariant 10, §Development Sequencing step 1, and `_content-plan.md` A9/B4.

Scope correction from exploration: there is no `agh.network.v0` NATS subject constant. Do not invent one. The existing `agh.soul.*`, `agh.heartbeat.*`, direct-room, envelope, and future-profile values are real identifiers and must be hard-cut in this task.

The wire-class gate covers live source, manifests/YAML, runtime fixtures, protocol RFCs/docs, active web/UI sources and their tests. It excludes only deferred historical/editorial content with an explicit later owner and `.compozy/tasks/**`; there is no allowlist for a live old wire identifier.

### Relevant Files

- `internal/network/envelope.go:12` — `ProtocolV0 = "agh-network/v0"`, the single definition site
- `internal/network/manager.go:19-20` — `RuntimePeerID = "agh.runtime"` and `runtimePeerSessionID`
- `internal/network/capability_brief.go:11`, `capability_catalog.go:12-14`, `manager_logging.go:145-148,177,182`, `greet_summary.go:41` — the eight `agh.*` ext keys at write and read sites
- `internal/api/core/network.go`, `internal/cli/network_delivery_output.go` — API/CLI readers and renderers of renamed envelope keys
- `internal/soul/{soul,persistence}.go`, `internal/heartbeat/heartbeat.go` — real branded digest/wake identifiers
- `internal/store/network_direct_identity.go` and `network_conversation_types_test.go` — persisted direct-room derivation prefix and canonical vector
- `internal/task/lease.go`, `internal/redact/{patterns,redact}.go`, `internal/api/contract/json_safety.go`, `internal/store/network_claim_token.go`, `internal/cli/client_tools.go`, `internal/situation/task_context_redaction.go`, and `internal/task/starvation_events.go` — raw claim generation, case-insensitive detection, redaction marker, and public-surface leak fences
- `internal/loop/dsl/types.go:1,12,22`, `codec.go:10,26`, and `internal/api/contract/loops.go:324` — actual `agh.loop/v1` declaration, parser/serializer text, and public DTO documentation
- `.compozy/loops/*/loop.yaml`, `extensions/dev-cycle/loops/*/loop.yaml` — embedded/local loop `apiVersion` literals after task 01's home hard cut
- `docs/rfcs/003_agh-network-v0.md`, `004_agh-network-v1.md` (+ `001`, `002`, `005` bodies) — protocol documents
- `packages/site/lib/__tests__/protocol-rfc-hard-cut.test.ts` — asserts old-string presence and pins RFC filenames
- `packages/site/content/protocol/`, `content/runtime/core/network/*`, and RFC 003–005 — protocol, direct-room, and future-profile docs
- `web/src/systems/loops/{lib,mocks,components}/**` and `packages/ui/src/components/**/stories/**` — active loop version rendering, fixtures, and UI contracts
- `docs/_memory/glossary.md` — protocol vocabulary read by the site test
- `skills/agh/**` — official runtime network/loop protocol guidance; directory/name hard cut remains task 04

### Dependent Files

- `internal/extension/loop_resources_test.go:104`, `internal/cli/loop_test.go:606`, `internal/daemon/{loop_run_events_e2e_integration,loop_resources,loop_pinning,task_runtime}_test.go`, `internal/api/contract/contract_test.go:190`, `internal/loop/{compiler,resource_spec,source_store}_test.go` — `apiVersion` fixtures
- `internal/testutil/acpmock/testdata/network_collaboration_fixture.json` — persisted network fixture
- Existing claim-token suites under `internal/{task,redact,api,cli,store,situation,diagnostics,support,transcript,tools}/**` and `web/e2e/**` — generation, case-insensitive redaction, transport/JSON safety, and forbidden-needle coverage; update in place rather than adding a duplicate suite
- `internal/update/github.go:154` — user-agent string `agh/<version>` (rename with the release identity in task 10, not this wire-class task)

### Related ADRs

- No ADR governs the wire rename directly; the brand and protocol-naming decisions are brief-locked (round 1, decision 2). The future standalone-protocol-brand trigger stays documented in the brief.

## Deliverables

- `compozy-network` wire-family identities with renamed peer id, envelope/digest prefixes, direct-room derivation, and future profiles
- `compozy.loop/v1` apiVersion across DSL, DTO, YAML, and fixtures
- `compozy_claim_` raw token identity with case-insensitive leak protection and unchanged `claim_token` / `claim_token_hash` field contracts
- Renamed RFC files with swept bodies and an updated site hard-cut spec passing in the same commit
- Swept site protocol content
- Wire-class grep gate at zero, `make verify` and the site test lane green
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from [`_tests.md`](_tests.md), the test contract — read each ID's full definition there before writing tests.

- [x] UT-059 — network subject/envelope constants: protocol version string and `compozy.*` envelope ext keys (update the existing transport/envelope suites; co-ship per Safety Invariant 10). Extend the owning soul/heartbeat/direct-room and HTTP/UDS/CLI suites for their real renamed values rather than creating duplicate regression suites. Update the existing claim-token generator/redaction/transport suites in place so the new raw prefix remains fenced everywhere; do not rename the public field names.

### Web/Docs Impact

- `web/`: `web/src/systems/loops/{lib,mocks,components}/**` renders and seeds the serialized loop document, so its source/tests/fixtures must co-ship with `compozy.loop/v1`. Checked `web/src/systems/network/**` and generated types: no network DTO field rename is expected. If generated types shift, regenerate them in this change.
- `packages/site`: `content/protocol/**`, `content/runtime/core/network/**`, `lib/__tests__/protocol-rfc-hard-cut.test.ts`, plus any protocol illustration referencing the wire string.
- QA impact: reset to `untested` every `docs/qa/scenarios/*.md` whose `entry_points` cite network channel/peer commands, the protocol version string, or claim-token forbidden needles; add content-addressed `untested` files for the renamed protocol and raw-token identities.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: protocol docs and the RFC set are extension-facing contracts for anyone implementing the protocol outside the daemon. Envelope/digest/direct-room identifiers are renamed wholesale; no hook, bundle, or skill surface changes.
- Agent manageability: `compozy network` verbs keep their shape; the protocol version reported by structured output changes value. Deterministic errors and status discovery are unchanged.
- Config lifecycle: no `config.toml` key changes — checked surfaces: `[network]` section keys, channel configuration; reason: the rename touches wire values, not configuration key names.

### AGH Impact Audit

- Native tools: no `agh__*` ToolID or ToolsetID change; checked descriptor catalogs, capability gates, and hosted MCP. Network tool results expose renamed wire values and must remain truthful.
- Extensibility and hooks: protocol RFCs/docs, envelope keys, digest prefixes, direct-room derivation, raw claim-token identity/redaction, and extension-facing network fixtures change; checked extensions, hooks, skills/capabilities, tools/resources, bundles, registries, bridge SDKs, and MCP sidecars.
- Workspace data isolation: protocol envelopes and direct-room derivation remain workspace-qualified. Update HTTP/UDS/CLI paths and fixtures together so workspace_id propagation, list/read/cache/SSE/event paths do not serialize an old identifier or cross-workspace value.
- Official AGH skill: keep the `skills/agh/` directory/name until task 04, but update all live network, peer, envelope, direct-room, and loop-apiVersion guidance here so the protocol contract co-ships.

## Success Criteria

- Every assigned test case implemented and passing
- Zero old wire-family strings, peer IDs, envelope/digest/direct-room prefixes, raw `agh_claim_`/`AGH_CLAIM_*` identities, or `agh.loop/v1` literals in the class grep gate scope
- `protocol-rfc-hard-cut.test.ts` passes against the renamed RFC filenames and the new wire string in the same commit
- `make verify` and the Bun site lane both green with no matcher asserting a renamed value

## Completion Evidence

```text
VERIFICATION REPORT
-------------------
Claim: Task 03 wire protocol and protocol-document hard cut is complete
Command: rtk make verify
Executed: 2026-07-26T15:22:06-03:00 through 2026-07-26T15:32:23-03:00, after all source changes
Exit code: 0
Output summary: CodegenCheck, installer, Bun lint/typecheck/test, Web build, Go fmt/lint/race tests/build, and boundaries passed; the site lane separately passed 50 files and 248 tests
Warnings: Existing non-blocking Vite chunk/plugin and Base UI uncontrolled-slider advisories; zero lint warnings
Errors: none
Contract parity: PASS — _brief.md decision 2; _techspec.md Safety Invariant 10 and delete targets; _content-plan.md A9/B4; _tests.md UT-059; task requirements
Visual contract: n/a — no named visual reference found; implementation capture qa/screenshots/task_03/loop-editor-dsl-compozy-version.png and teardown.json clean=true
Verdict: PASS
```
