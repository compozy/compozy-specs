---
status: completed
title: Connectivity provider seam and bundled Tailscale provider
type: backend
complexity: high
---

# Task 3: Connectivity provider seam and bundled Tailscale provider

## Overview

Opens connectivity as an extension point and ships its first implementation: a public `connectivity.provider` provide surface, a daemon-side consumer and supervisor, the challenge-based endpoint verification that proves an advertised address reaches this daemon's assigned tier, and a bundled first-party provider that embeds Tailscale so the operator installs nothing. Building the contract and its first implementation in one slice validates the contract immediately.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- The `connectivity.provider` surface MUST be public (third parties may implement it), not bundled-only like `bridge.adapter`.
- Verification MUST be a tier-scoped challenge: a nonce served only on the assigned tier listener and echoed by the advertised endpoint. A reachability probe alone is not acceptable.
- Verification MUST enforce an HTTPS scheme allowlist for public-tier endpoints, mandatory TLS validation (never `InsecureSkipVerify`), no redirect following, a bounded response body, an explicit deadline on a dedicated client (never `http.DefaultClient`), and an SSRF guard rejecting inward-resolving public endpoints.
- An endpoint MUST be advertised only after verification succeeds; an unverified endpoint marks the provider degraded and advertises nothing.
- Provider install source and control digest MUST be re-derived from the live extension registry on every enable and every boot; a mismatch fails closed and requires re-consent.
- Workspace-scoped extension sources MUST be rejected for this surface.
- Third-party providers MUST require digest confirmation before any provider code influences reachability; manifests validate before execution.
- Provider failure MUST degrade the affected reachability to a safe state — never half-exposed — with bounded teardown and supervised restart.
- At most one active provider per tier MUST be enforced; two providers claiming one tier require explicit operator selection.
- No extension→daemon Host API method may be added; providers report through the daemon-initiated status call, leaving the closed Host API set unchanged at 95 methods.
- The bundled provider MUST use the operator's own account and MUST NOT depend on a separately installed client binary.
- Protocol behavior MUST be covered by a real-subprocess conformance fixture, not only by an in-memory fake.
</requirements>

## Subtasks

- [x] 3.1 Add the `connectivity.provider` capability and its daemon→provider service methods to the extension protocol
- [x] 3.2 Implement the daemon-side connectivity consumer that resolves a provider by capability and dispatches service calls
- [x] 3.3 Implement the provider supervisor: enable, disable, health supervision, bounded teardown, and safe degradation
- [x] 3.4 Implement the endpoint verification protocol with the challenge, scheme allowlist, TLS, redirect, size, deadline, and SSRF rules
- [x] 3.5 Implement trust re-derivation from the live extension registry with digest confirmation and re-consent on mismatch
- [x] 3.6 Reject workspace-scoped sources and enforce one active provider per tier
- [x] 3.7 Build the bundled Tailscale provider package with embedded manifest and provider runtime, establishing both tiers
- [x] 3.8 Register the bundled provider's re-exec entry point and enroll it on boot
- [x] 3.9 Add the connectivity provider scaffold template and SDK authoring helpers
- [x] 3.10 Build the real-subprocess conformance fixture covering negotiation, malformed payloads, crash, teardown, and redaction

## Implementation Details

Follow the TechSpec sections *Endpoint Verification Protocol*, *Core Interfaces* (ConnectivitySource, ProviderSupervisor), and *Extensibility Integration Plan*. Adding a provide surface is a single-file change in the protocol package that auto-propagates to manifest validation and contract derivation. The consumer mirrors the existing watch-source/model-source pattern; the bundled package mirrors the bundled dev-cycle extension (embedded manifest plus a re-exec provider runtime).

### Relevant Files

- `internal/extensionprotocol/capabilities.go` — the closed provide-surface set and its capability→service-method map; adding the surface here propagates automatically
- `internal/extension/contract/capabilities.go` — public/private derivation for provide surfaces
- `internal/extension/manifest.go`, `internal/extension/manifest_load.go`, `internal/extension/manifest_permissions.go` — manifest schema and the validate-before-execution gates
- `internal/extension/watch_source.go`, `internal/extension/model_source.go`, `internal/extension/contract/model_source.go` — the daemon-side consumer trio to mirror as `gateway_source.go` + `contract/gateway_source.go`
- `internal/extension/contract/sdk.go`, `internal/extension/contract/describe.go` — SDK declaration and describe-mode advertisement the new surface must join
- `internal/extension/scaffold_templates/loop-watch-source-go/` — the scaffold template to mirror for a connectivity-provider template
- `internal/extension/extension_validation.go`, `internal/extension/manifest_normalize.go` — surface-specific validation hooks
- `internal/extension/tool_runtime.go` — how a service method is resolved to a live process and dispatched
- `internal/extension/registry_install.go` — install-source validation, including the `bridge.adapter` gate as the precedent for surface-specific rules
- `internal/extension/capability.go`, `internal/extension/capability_resource_policy.go` — install-source tiers and per-source ceilings
- `internal/extension/manager_supervision.go`, `internal/extension/manager_failure_state.go` — supervision, backoff, and crash-loop disable semantics
- `internal/subprocess/handshake.go`, `internal/subprocess/transport.go` — the JSON-RPC/stdio protocol and initialize handshake
- `extensions/dev-cycle/embed.go`, `extensions/dev-cycle/extension.json`, `extensions/dev-cycle/install.go`, `extensions/dev-cycle/runtime.go` — the bundled-extension packaging pattern
- `internal/cli/internal.go` — the re-exec switch that runs a bundled provider runtime
- `internal/daemon/boot_automation_extensions.go` — bundled enrollment plus manager startup
- `internal/extension/scaffold.go` and its template directory — where the new provider scaffold lands
- `internal/outboundpolicy/http_client.go` — existing outbound client construction with explicit timeouts and TLS minimums

### Dependent Files

- `internal/gateway/` — the supervisor and verification live here and are consumed by the reconciler from task 01
- `go.mod` — the embedded Tailscale dependency is added via `go get`
- `internal/events/` — provider state-change and endpoint verification events registered with this task's surfaces
- `packages/site/content/docs/` — provider authoring and connectivity documentation

### Related ADRs

- [ADR-009: `connectivity.provider` surface, challenge-based verification, registry-derived trust](adrs/adr-009.md) — the surface, verification, and trust-freshness contract
- [ADR-003: Exposure policy in core; connectivity mechanisms as provider extensions](adrs/adr-003.md) — the policy/mechanism split
- [ADR-001: First-party managed connectivity without hosted infrastructure](adrs/adr-001.md) — operator-owned accounts, no Compozy infrastructure

## Deliverables

- Public `connectivity.provider` provide surface with its service methods and manifest validation
- Daemon-side connectivity consumer, provider supervisor, and challenge-based endpoint verification
- Registry-derived trust with digest confirmation, re-consent on mismatch, and workspace-source rejection
- Bundled first-party Tailscale provider establishing both tiers with zero operator installation
- Provider scaffold template and SDK authoring helpers for third parties
- Real-subprocess conformance fixture for the provider protocol
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-060, UT-061, UT-062 — enable ordering, abandoned authorization, unauthorized enable without partial exposure
- [x] UT-063, UT-064 — challenge success and wrong-tier/no-nonce failure
- [x] UT-065, UT-066, UT-067, UT-068, UT-069, UT-070 — scheme allowlist, TLS validation, redirect rejection, body bound, deadline on a dedicated client, SSRF guard
- [x] UT-071 — unverified endpoint never advertised and provider marked unhealthy
- [x] UT-072, UT-073 — trust re-derivation from the live registry and stale-digest fail-closed
- [x] UT-074, UT-075, UT-076, UT-077 — digest confirmation, re-consent on expanded control, out-of-role refusal, workspace-source rejection
- [x] UT-078, UT-079, UT-080 — outage degradation, bounded teardown, one active provider per tier
- [x] UT-081, UT-082, UT-083, UT-084, UT-085 — real-subprocess conformance: negotiation, malformed payloads, unimplemented method, crash/teardown, output redaction
- [x] IT-030, IT-031, IT-032 — real provider enable through the extension seam, wrong-tier endpoint failing end to end, extension update invalidating a confirmed digest
- [x] UT-162, UT-163, UT-164 — public DNS isolation, staged provider reuse, and automatic recovery after an initial endpoint-proof failure
- [x] UT-165 — provision the selected Tailscale HTTPS certificate before opening the public Funnel listener
- [x] UT-166 — separate the provider activation deadline from the endpoint-proof timeout
- [x] UT-167 — authenticate the public DNS-over-TLS resolver and reject an invalid certificate

## Success Criteria

- Every assigned test case implemented and passing
- A test proves an endpoint pointed at the wrong tier fails verification and is never advertised
- A test proves a stale digest after an extension update fails closed and demands re-consent
- The bundled provider brings up both tiers on a machine with no separately installed client
- The closed Host API method set is unchanged; `make codegen-check` reports no drift
- Provider crash and teardown never leave a half-exposed surface
