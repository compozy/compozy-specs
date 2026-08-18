---
status: completed
title: Tier listeners and device authentication
type: backend
complexity: critical
---

# Task 2: Tier listeners and device authentication

## Overview

Makes the daemon authenticably reachable: the composition root gains a tier-listener factory that builds per-tier HTTP servers with their own route matrices, and `internal/gateway` gains the device-session model — credentials, pairing, stream tickets, and a device-keyed connection registry — behind a new device-auth middleware. This is the security spine of the feature: after this task a paired device can use the full product over the private tier, and revocation actually severs live streams.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- The daemon MUST NOT bind any non-loopback address; tier listeners bind loopback only and are constructed by `internal/daemon`, never by `internal/gateway`.
- Tier MUST be a property of the listener, never a request header: each tier listener is built with its own route matrix so an absent route physically does not exist there.
- The pairing mint and redeem routes MUST exist on the private tier only, and MUST NOT be registered on the public tier under any configuration, including with the operator UI consented.
- Tier listeners MUST grant no loopback trust: every request either authenticates as a device session or self-verifies.
- Device credentials MUST be 256-bit random with the documented prefixes, stored only as a SHA-256 hash under a unique index, and compared constant-time.
- Revocation MUST bump the durable revoke epoch and cancel every live stream for that device via the connection registry **before returning**; in-flight mutations MUST revalidate the epoch inside their transaction and fail rather than commit partially.
- Stream tickets MUST be single-use and short-TTL; every reconnect MUST acquire a fresh ticket, and no credential may appear in a stream URL.
- The browser MUST receive an httpOnly, Secure, SameSite cookie at redeem and never a raw credential; credential-bearing responses set `no-store` and `no-referrer`.
- Pairing artifacts MUST be in-memory, single-use, bounded by `max_pending`, and TTL-expired; redeem MUST be atomic so exactly one redeemer wins a race.
- Existing webhook routes MUST stay registered ahead of the device-auth gate so their independent verification is preserved.
- The UDS superset (agent kernel, hosted MCP, MCP Host API, resource mutation) MUST NOT be mounted on any tier listener.
- Route registration, contract DTOs, OpenAPI, and generated TypeScript MUST co-ship in this task.
</requirements>

## Subtasks

- [x] 2.1 Add `WithSurfaceSet` and `WithDeviceAuth` options to the HTTP server and define the per-tier route matrices, including SPA behavior per tier
- [x] 2.2 Add the daemon-owned tier-listener factory, publish resolved ports back into policy before advertisement, and order startup/shutdown
- [x] 2.3 Implement the device service: credential generation, hashing, constant-time verification, rename, revoke with epoch, and listing
- [x] 2.4 Implement bounded in-memory pairing artifacts with atomic single-use redeem and TTL expiry
- [x] 2.5 Implement the device-keyed connection registry and integrate register/deregister into every authenticated stream handler
- [x] 2.6 Implement single-use stream tickets and the ticket-based SSE/WebSocket handshake
- [x] 2.7 Implement the device-auth middleware accepting bearer or cookie, with per-source failed-auth rate limiting exempting local access
- [x] 2.8 Add the gateway service interface, shared handlers, contract DTOs, and `StatusForGatewayError` mapping
- [x] 2.9 Register gateway management routes on the private tier and the UDS management subset, keeping transport parity
- [x] 2.10 Regenerate OpenAPI and TypeScript, and document the new endpoints

## Implementation Details

Follow the TechSpec sections *Core Interfaces*, *Credential Ownership and Stream Flow*, *API Endpoints and Tier Route Matrices*, and *Safety Invariants* (1–3, 10–14). `WithResourceOperatorAuth` is the exact injection-point pattern to copy for `WithDeviceAuth`: option → server field → handler config → handlers → applied at route registration, with a construction-time validation guard. Handlers are shared `BaseHandlers` methods so HTTP and UDS stay in parity.

### Relevant Files

- `internal/api/httpapi/bridge_routes.go` — the route-group file pattern to copy for a new `gateway_routes.go` (mirror it in `internal/api/udsapi/`)
- `internal/api/httpapi/server_network_options.go` — the per-domain option file pattern for a new `server_gateway_options.go`
- `internal/api/core/marketplace.go` (+ `marketplace_interfaces.go`, `marketplace_paths.go`) — the handler-set pattern to copy for `gateway.go`, `gateway_payloads.go`, `gateway_interfaces.go`
- `internal/api/spec/operation_registry.go` — `buildOperationRegistry()`; register the gateway operations here
- `internal/api/spec/registry_vault.go` — the compact registry file pattern for a new `registry_gateway.go`
- `internal/api/httpapi/server.go` — `Option` type, `Server` fields, and `WithResourceOperatorAuth` as the pattern for `WithDeviceAuth`/`WithSurfaceSet`
- `internal/api/httpapi/server_setup.go` — middleware install order and construction-time validation guards
- `internal/api/httpapi/routes.go` — route registration order; webhooks and the MCP OAuth callback register ahead of the guards and must stay that way
- `internal/api/httpapi/middleware.go` — CORS, browser/origin protection, and the loopback guards this task supersedes for tier listeners
- `internal/api/httpapi/static.go` — SPA fallback behavior, which must be tier-aware
- `internal/api/httpapi/handlers.go` — handler struct and per-route guard helpers
- `internal/api/httpapi/server_start.go` — `listenConfig.Listen` and port-resolution for `:0`
- `internal/api/udsapi/routes.go` — UDS registration and the superset that must never be exposed
- `internal/api/core/base_handlers.go` — `BaseHandlers` and its config
- `internal/api/core/errors.go` — `RespondError` plus the per-domain `StatusFor*` mappers to extend
- `internal/api/core/sse.go` — `PrepareSSE` and stream write helpers where registry hooks attach
- `internal/api/contract/status.go` — status payload to extend with gateway posture
- `internal/api/spec/` — operation registry entries and auth metadata for the new routes
- `internal/daemon/daemon_defaults.go`, `internal/daemon/server_options.go` — server factory defaults and option assembly for the tier factory
- `internal/mcp/serve_http.go` — the existing constant-time bearer handler as a verification reference

### Dependent Files

- `internal/api/httpapi/transport_parity_integration_test.go`, `internal/api/udsapi/transport_parity_integration_test.go` — parity suites gain the gateway management surface
- `openapi/compozy.json`, `web/src/generated/compozy-openapi.d.ts` — regenerated in this task
- `internal/cli/daemon_status.go` — status rendering picks up gateway posture
- `packages/site/content/docs/` — endpoint reference for the new routes

### Related ADRs

- [ADR-007: Daemon-owned tier listeners](adrs/adr-007.md) — listener ownership, route matrices, port publication
- [ADR-008: Opaque revocable credentials, stream tickets, connection-registry revocation](adrs/adr-008.md) — credential format, cookie/bearer split, revocation mechanism
- [ADR-004: Public tier surface set](adrs/adr-004.md) — why pairing never exists publicly
- [ADR-002: Single-operator identity with per-device pairing](adrs/adr-002.md) — device model and root of trust

## Deliverables

- Per-tier HTTP listeners with distinct route matrices, built by the composition root and torn down in order
- Device service with credentials, pairing, tickets, revocation epoch, and the connection registry
- Device-auth middleware accepting bearer or cookie, with rate limiting and local exemption
- Gateway management endpoints on the private tier and the UDS subset, in transport parity
- Contract DTOs, OpenAPI, and generated TypeScript regenerated in the same change
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-030, UT-031, UT-046, UT-049 — credential format, constant-time verification, revoked-credential rejection, client-local credential file
- [x] UT-032, UT-033, UT-034, UT-035, UT-036 — pairing expiry, malformed rejection without an existence oracle, atomic single-use redeem, bounds, local-mint semantics
- [x] UT-037, UT-038, UT-039, UT-043, UT-044 — device listing, ordering, idempotent revoke, self-revoke, last-device revoke
- [x] UT-040, UT-041, UT-042 — revoke epoch plus registry cancellation and pre-commit revalidation
- [x] UT-045 — per-source failed-auth rate limiting with local exemption
- [x] UT-047, UT-048 — single-use stream tickets and fresh-ticket-per-reconnect
- [x] UT-051, UT-052 — deterministic not-found and valid empty device list
- [x] UT-050, UT-106, UT-107 — gateway type→DTO mapping and error→status/code mapping including the masked/raw split
- [x] IT-001, IT-003, IT-006, IT-007 — fail-closed listener behavior, concurrent transitions across transports, disable terminating sessions, status parity
- [x] IT-010, IT-011, IT-012, IT-013, IT-014 — loopback-only binding, redeem present privately and absent publicly, public inventory, no loopback trust
- [x] IT-015, IT-016, IT-017, IT-018 — consented public UI reachability, port publication before advertisement, uniform probing responses, load isolation
- [x] IT-020, IT-021, IT-022, IT-023, IT-024, IT-025 — pairing race, HTTP/UDS device parity, live-stream termination on revoke, aborted in-flight mutation, ticket handshake, browser cookie flow
- [x] IT-073 — paired device reaches the full product privately while a reachable-unpaired client sees only the gate

## Success Criteria

- Every assigned test case implemented and passing
- A test proves the pairing redeem route is absent from the public listener in every configuration
- A test proves revoking a device terminates an open SSE connection before the call returns
- `make codegen-check` reports no drift after OpenAPI/TypeScript regeneration
- Transport parity suites cover every new gateway management operation
- No UDS-only route is reachable on either tier listener
