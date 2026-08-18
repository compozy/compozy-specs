---
status: completed
title: "Self-audit, native tool, and cross-surface assurance"
type: backend
complexity: medium
---

# Task 6: Self-audit, native tool, and cross-surface assurance

## Overview

Closes the loop on "am I safe?" and "can an agent fix this?": a security self-audit that reports exposure posture with ranked, actionable findings, a `compozy__gateway` native tool so in-session agents can inspect and repair their own daemon, and the suite-level assurances that only become verifiable once every gateway surface exists — complete event-family registration, native catalog drift, and a cross-surface secret scan.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- The audit MUST run against any posture, including a fully local-only daemon, and MUST report an explicit no-findings result distinct from "did not run".
- Findings MUST carry stable identities, severity, and concrete remediation so an agent can branch on them and a re-run can confirm the fix cleared.
- The audit MUST be side-effect-free and cheap enough for an agent to run in a loop.
- A provider outage MUST surface as a finding, never as an audit failure.
- The native tool MUST expose read operations (status, audit) unconditionally and gate mutations by permission mode, with a complete descriptor, binding, dependency, and availability path plus a regenerated catalog with no drift.
- Every `gateway.*` event MUST be registered with its component and severity metadata, and every gateway event MUST be durably appended before it is published to live consumers.
- Every gateway route MUST appear in the API operation registry with auth metadata, and HTTP and UDS registrations MUST agree.
- Remote actions MUST be attributed using the canonical actor fields — no new event column and no migration.
- No raw claim token, device credential, pairing artifact, or stream ticket may cross any surface: HTTP tiers, UDS, CLI output, SSE frames, events, or memory. Only hash, ref, or truncated forms are permitted.
- Redaction MUST be asserted by scanning emitted bytes, not by reviewing code paths.
</requirements>

## Subtasks

- [x] 6.1 Implement the auditor: posture collection across exposure, auth, devices, providers, and ingress
- [x] 6.2 Define the finding catalog with stable ids, severity ordering, and concrete remediation text
- [x] 6.3 Expose the audit through the private tier, the UDS subset, and a CLI verb with structured output
- [x] 6.4 Author the `compozy__gateway` native tool descriptor with input and output schemas and risk flags
- [x] 6.5 Wire the tool binding, dependencies, and availability diagnostics, and regenerate the native tool catalog
- [x] 6.6 Complete gateway event registration: component, names, severity metadata, and payload redaction mapping
- [x] 6.7 Verify durable-append-before-publish for every gateway event
- [x] 6.8 Attribute remote state-changing actions using the canonical actor fields and confirm they remain queryable
- [x] 6.9 Build the cross-surface secret scan covering every gateway-reachable output path
- [x] 6.10 Update the official Compozy skill with gateway verbs, the operation matrix, native tool usage, and error codes

## Implementation Details

Follow the TechSpec sections *Monitoring and Observability*, *Agent Manageability Plan*, and *Safety Invariant 18*. Event registration follows the existing component-plus-registry pattern: a component constant, a per-domain names file, and registry entries built with the existing severity and global wrappers, all asserted by the registry suite. The native tool follows the existing descriptor and catalog pattern, whose generated JSON is drift-checked.

### Relevant Files

- `internal/events/components.go` — component constants; the gateway component is added here
- `internal/events/extension.go` — a per-domain event-names file to mirror for gateway events
- `internal/events/registry_base.go` — registration entries built with the severity and global wrappers
- `internal/events/registry_test.go` — the canonical registry suite that asserts exact membership
- `internal/events/registry_metadata.go` — per-event metadata shape
- `internal/tools/builtin/descriptors.go` — descriptor definitions
- `internal/tools/builtin/catalog.go` — catalog assembly
- `internal/tools/builtin/marketplace.go` — the minimal tool-file pattern; `internal/tools/builtin/network.go` is the full-featured one to mirror for a richer surface
- `internal/tools/builtin/toolsets.go` — toolset membership for the new tool
- `internal/daemon/native_bridge_catalog_tools.go` — the daemon-side native tool handler pattern for a new `native_gateway_tools.go`
- `internal/daemon/native_tools.go`, `internal/daemon/native_tools_dependencies_builder.go` — tool registration and dependency wiring
- `internal/api/spec/operation_registry.go` — `buildOperationRegistry()` must include the gateway operations for the registry assertion to pass
- `magefiles/openapi_artifacts.go` — the OpenAPI artifact generator behind `make codegen`
- `internal/tools/builtin/testdata/native-tool-catalog.json` — the generated catalog that must not drift
- `internal/tools/builtin/builtin_test.go` — the canonical native tool suite
- `internal/api/spec/` — operation registry entries and auth metadata for gateway routes
- `internal/api/core/errors_diagnostic_payload.go` — the existing redaction path that gateway secrets extend
- `internal/observe/observer_event_persistence.go` — durable append before live publication
- `internal/diagnostics/` — redaction helpers used across status and diagnostics output
- `skills/compozy/` — the bundled official skill describing agent-facing surfaces

### Dependent Files

- `internal/gateway/` — the auditor and its finding catalog live here
- `internal/api/httpapi/`, `internal/api/udsapi/` — audit endpoint registration in both transports
- `internal/cli/` — the audit verb and its structured output
- `openapi/compozy.json`, `web/src/generated/compozy-openapi.d.ts` — regenerated for the audit endpoint
- `packages/site/content/docs/` — agent-facing documentation of the audit and native tool

### Related ADRs

- [ADR-006: Fail-closed exposure with agent-operable self-audit](adrs/adr-006.md) — the audit contract and discoverability requirement
- [ADR-005: Remote client methods and the bounded operation matrix](adrs/adr-005.md) — the matrix the native tool and skill publish
- [ADR-008: Opaque revocable credentials](adrs/adr-008.md) — what must never appear unredacted

## Deliverables

- Security self-audit with a stable finding catalog, exposed over the private tier, UDS, and CLI with structured output
- `compozy__gateway` native tool with descriptor, binding, availability diagnostics, and a regenerated drift-free catalog
- Complete gateway event registration with component, severity, and redaction metadata, appended durably before publication
- Canonical-actor attribution for remote actions, queryable through existing observability paths
- Cross-surface secret scan proving no raw credential or claim token escapes any gateway surface
- Updated official Compozy skill covering the agent path
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-120, UT-121, UT-122 — no raw claim token anywhere, credentials and tickets only in redacted form, wrong-field secrets redacted before logging
- [x] UT-130, UT-131, UT-132, UT-133 — audit content and ranking, provider outage as a finding, explicit no-findings result, side-effect-free repeatability
- [x] UT-140 — canonical actor attribution distinguishing remote from local actions
- [x] UT-150, UT-151 — complete gateway event registration and durable append before publish
- [x] UT-152 — every gateway route in the operation registry with auth metadata and HTTP/UDS agreement
- [x] UT-153, UT-154 — native tool descriptor resolution through binding, dependencies, and availability; generated catalog without drift
- [x] IT-080, IT-081 — byte-scan of every live gateway surface for raw claim tokens and for credentials, artifacts, and tickets
- [x] IT-082 — gateway events queryable by actor through existing observability paths
- [x] IT-083 — HTTP and UDS gateway operations match their registry entries including auth metadata

## Success Criteria

- Every assigned test case implemented and passing
- An audit finding can be cleared by following its own remediation text, verified by a re-run
- The byte-scan suite fails if any raw credential shape reaches any gateway-reachable output
- `make codegen-check` reports no native catalog or contract drift
- The registry suite fails if a `gateway.*` event is emitted without registration
- An agent can inspect and repair daemon exposure using only structured surfaces, with no web UI
