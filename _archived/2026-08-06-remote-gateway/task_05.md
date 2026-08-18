---
status: completed
title: "Remote CLI profiles, operation matrix, and SSH connect"
type: backend
complexity: high
---

# Task 5: Remote CLI profiles, operation matrix, and SSH connect

## Overview

Gives the terminal — and the agents that drive it — a way to operate a daemon that is not on this machine. The CLI client gains a connection target abstraction (local socket versus remote HTTPS with a credential), named connection profiles with client-local credential storage, an enforced local-only/remote-capable operation matrix, and an SSH connection method that brings up or reuses a daemon on any host the operator can already reach.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- The client MUST resolve a connection target and attach its credential at the single existing request chokepoint; the local socket path MUST behave exactly as today.
- Remote-capable operations MUST be limited to the published matrix; the agent kernel, hosted MCP, MCP Host API, and resource mutation stay UDS-only and MUST NOT be reachable through any profile.
- A verb bound to a local-only operation MUST fail against a remote profile with a deterministic error rather than acting on the wrong machine.
- The gateway MUST own no queue, scheduler, or claim primitive: no remote path may claim, heartbeat, complete, fail, or release a task run.
- Profile metadata MUST live in the operator-global config file while the credential lives in a client-local file with owner-only permissions; removing a profile MUST delete both.
- Reachability failures, authentication failures, and local-only refusals MUST have distinct, stable error identities so agents can branch on them.
- Work started from a remote client MUST be detached and observable after reconnect.
- Streaming MUST work over a remote profile, acquiring a fresh stream ticket per connection and per reconnect.
- The SSH method MUST use the operator's existing SSH credentials and configuration, MUST fail closed on a changed host key, MUST bind its local forward to loopback only, and MUST stop only daemons it started.
- The SSH method MUST verify remote compatibility before adopting a running daemon and MUST surface absent-installation and version-mismatch as distinct deterministic outcomes.
- `compozy open` MUST become scheme- and host-aware for the active profile.
</requirements>

## Subtasks

- [x] 5.1 Introduce a connection-target abstraction in the client and branch transport construction between local socket and remote HTTPS
- [x] 5.2 Attach the credential at the single request chokepoint and mirror the change for the streaming and WebSocket dial paths
- [x] 5.3 Implement profile storage: config-file metadata plus a client-local credential file with owner-only permissions, portable encrypted export/import, and clean removal
- [x] 5.4 Resolve the active profile during runtime context construction, replacing socket-only resolution
- [x] 5.5 Add the `gateway`, `pair`, `device`, and `connect` command groups with structured output
- [x] 5.6 Implement and enforce the local-only/remote-capable operation matrix with its deterministic refusal
- [x] 5.7 Make `compozy open` scheme- and host-aware for the active profile
- [x] 5.8 Implement the SSH connection method: host resolution, remote launch or verified reuse, local loopback forward, and scoped teardown
- [x] 5.9 Implement SSH failure taxonomy for absent installation, version mismatch, changed host key, unreachable host, and non-default remote home
- [x] 5.10 Publish the operation matrix in CLI help and ship the CLI reference documentation for the new verbs

## Implementation Details

Follow the TechSpec sections *Remote Operation Matrix*, *Credential Ownership and Stream Flow*, and *Agent Manageability Plan*. The client currently hardcodes a socket base URL and dials a Unix socket in one constructor; that constructor and the runtime-context resolver are the two seams to widen. Header attachment already happens in one request builder, which is where the credential joins the existing agent-identity headers. Use an existing multi-verb command group as the structural template.

### Relevant Files

- `internal/cli/client.go` — the hardcoded base URL constant and the daemon client interface
- `internal/cli/client_settings_vault.go` — the client constructor that dials the Unix socket
- `internal/cli/client_transport.go` — the single request-building chokepoint where headers are set and SSE is decoded
- `internal/cli/client_window_manager_stream.go` — the WebSocket dial that needs the same target switch
- `internal/cli/root_runtime.go` — runtime context and client construction from configuration
- `internal/cli/root.go`, `internal/cli/root_defaults.go` — command registration and dependency defaults
- `internal/cli/vault.go` — the compact multi-verb command group template; `internal/cli/bridge.go` (+ `bridge_commands.go`, `bridge_output.go`) is the split-file shape if the gateway group outgrows one file
- `internal/cli/client_network.go`, `internal/cli/client_bridge_control.go` — the per-domain client method file pattern for a new `client_gateway.go`
- `internal/cli/client_test.go`, `internal/cli/cli_integration_test.go` — the canonical client and CLI integration suites these changes extend
- `internal/cli/open.go` — the hardcoded scheme to make profile-aware
- `internal/cli/config.go` — settable-key registration for connection profiles
- `internal/config/persistence.go` — overlay editing used to write profile metadata into the config file
- `internal/agentidentity/identity.go` — the existing identity headers the credential sits beside
- `internal/api/udsapi/routes.go` — the UDS-only superset that defines the local-only side of the matrix
- `internal/procutil/` — process-group signaling helpers relevant to SSH-launched process lifecycle

### Dependent Files

- `internal/gateway/` — device sessions created by CLI pairing appear in the device list
- `packages/site/content/docs/cli/` — generated CLI reference for the new verbs
- `internal/api/core/` — the deterministic error identities the CLI surfaces
- `web/` — unaffected by this task; the operation matrix is documented for the operator surface task

### Related ADRs

- [ADR-005: Remote client methods and the bounded remote operation matrix](adrs/adr-005.md) — the matrix and the SSH method
- [ADR-008: Opaque revocable credentials](adrs/adr-008.md) — client-local credential storage and stream tickets
- [ADR-010: Gateway state ownership](adrs/adr-010.md) — profile metadata in config, credential in a client-local file

## Deliverables

- Connection-target-aware client transport covering JSON, SSE, and WebSocket paths
- Named connection profiles with config metadata, client-local credential storage, and passphrase-protected transfer, created by pairing
- `gateway`, `pair`, `device`, and `connect` command groups with structured output and deterministic errors
- Enforced local-only/remote-capable operation matrix, published in help and documentation
- SSH connection method with launch-or-verified-reuse, loopback-only forward, scoped teardown, and a complete failure taxonomy
- Profile-aware `compozy open`
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-100, UT-101, UT-108 — reachability versus auth error distinction, duplicate profile conflict, target-aware request construction at the chokepoint
- [x] UT-102 — local-only operation refused deterministically against a remote profile
- [x] UT-103, UT-104, UT-105 — mid-stream loss outcome, valid-or-truncated structured output, revoked-profile isolation in a loop
- [x] UT-110, UT-111, UT-112, UT-113 — absent remote installation, version mismatch, changed host key, non-default remote home
- [x] IT-060, IT-061 — pairing creates profile, credential, and device row together; a copied profile is the same revocable identity
- [x] IT-062 — remote-capable verbs behave identically to local, including streaming
- [x] IT-063, IT-064 — UDS-only routes absent from tier listeners; no remote path can claim or heartbeat a run
- [x] IT-065, IT-066 — two clients on one daemon stay consistent; remote-started work survives disconnect and is observable on reconnect
- [x] IT-070, IT-071, IT-072 — SSH launch or verified reuse with scoped teardown, drop-and-resume behavior, loopback-only exposure with remote gateway state untouched
- [x] E2E-002 — pair a CLI profile, run remote work, disconnect and reconnect, observe progress, and hit the local-only refusal
- [x] E2E-004 — SSH connect to a harness remote, work locally, disconnect leaving no new reachable surface

## Success Criteria

- Every assigned test case implemented and passing
- A test proves no remote path can reach `ClaimNextRun` or any agent-kernel route
- Local CLI behavior over the Unix socket is unchanged, verified by the existing suites
- The operation matrix in CLI help matches the enforced behavior exactly
- SSH connect leaves no new reachable surface on either machine and stops only what it started
- Removing a profile deletes both its metadata and its credential file
