# Test Specification: Remote Gateway

Canonical test contract for the Remote Gateway. Companion to `_techspec.md`.
Derived from `_user_stories.md` (behavior) and `_techspec.md` (components).

> Revision 2 — rebuilt after peer-review round 1 blocker B-012. Every row now names its **invariant**, its **owning layer**, and the **canonical suite** it extends. Cross-layer duplication appears only where the layers fail differently, and that difference is stated in the row.

## Strategy

- **Frameworks/harness:** Go `testing`, `t.Run("Should …")` subtests, `-race` / `CGO_ENABLED=1`, table-driven. `t.Parallel` by default; **never** on cases that mutate process env or the shared home (L-002).
- **Fakes only at I/O boundaries:** a fake clock (TTL, epoch, generation ordering) and a fake HTTP origin for endpoint-verification cases. The connectivity provider protocol is covered by a **real subprocess conformance fixture** (`internal/extension/connectivity_conformance_test.go`), not by the in-memory fake — the in-memory `ConnectivitySource` is used only for supervisor-logic unit cases where the protocol is not the invariant under test (L-007).
- **Execution:** unit `make test`; integration `make test-integration` (`+integration`); E2E runtime `make test-e2e-runtime`; E2E web `make test-e2e-web`. The bundled `tsnet` provider runs only in a tagged manual lane, never in CI.
- **Conventions:** status code **and** body/code asserted on every endpoint case; sentinels asserted with `errors.Is`; no `_ =` error discards; redaction asserted by byte-scanning emitted output for raw credential shapes.

### Test placement (owning layers)

| Layer | Owns | Canonical suite |
| --- | --- | --- |
| Domain unit | Policy/state-machine logic, credential and pairing rules, registry mechanics, verification decisions, audit computation | `internal/gateway/*_test.go` |
| Transport unit | Error→status mapping, gateway-type → contract-DTO mapping | `internal/api/core/errors_test.go`, `internal/api/core/gateway_handlers_test.go` |
| Config unit | Defaults, validation, overlay, `config set` classification | `internal/config/gateway_test.go`, `internal/cli/config_test.go` |
| Integration | Wiring across packages: tier route matrices, middleware over a real listener, reconcile-after-crash, revocation across live streams, provider enable→verify→advertise, workspace authorization, CLI transport | `internal/api/httpapi/gateway_integration_test.go`, `internal/daemon/gateway_boot_integration_test.go`, `internal/gateway/integration_test.go` |
| Protocol conformance | Real provider subprocess: negotiation, malformed payloads, crash/teardown, redaction | `internal/extension/connectivity_conformance_test.go` |
| Schema | Migration fresh/reopen/ahead/integrity/equivalence | existing `internal/store/globaldb` migration suites |
| Registry | Event registration, API operation registry, native tool catalog | `internal/events/registry_test.go`, `internal/api/spec` parity suites, `internal/tools/builtin/builtin_test.go` |
| E2E | User journeys through the public surface | `test/e2e/runtime`, `test/e2e/web` |

## Coverage Matrix

Columns: **Invariant** (what must hold) · **Owning layer** · **IDs** · **Other layers** (only when a different failure mode justifies it).

### Exposure policy and state machine

| Source | Invariant | Owning layer | IDs | Other layers (distinct failure) |
| --- | --- | --- | --- | --- |
| US-003, INV-4 | Config cannot carry exposure intent; DB is sole authority | Domain unit | UT-010, UT-011 | Config unit UT-004 (a config key that could enable a surface must not exist) |
| US-003, INV-2 | Non-local exposure without auth active is refused with cause+fix | Domain unit | UT-012 | Integration IT-001 (refusal must also prevent the listener from serving, not just return an error) |
| US-003.EC-1 | Hand-edited forbidden config → local-only, refusal surfaced, no crash-loop | Integration | IT-002 | — |
| US-003.EC-2 | Public UI before pairing possible → ordered refusal | Domain unit | UT-013 | — |
| US-003.EC-3 | Removed legacy exposure semantics → refused, no compat interpretation | Domain unit | UT-014 | — |
| US-002.EC-4, INV-5 | Concurrent transitions fence on `generation`; stale effects discarded | Domain unit | UT-015, UT-016 | Integration IT-003 (two real callers via HTTP+UDS race one transition) |
| INV-6 | Effect order with compensation leaves no externally reachable intermediate | Domain unit | UT-017, UT-018 | Integration IT-004 (fault injection at each of the 5 effect boundaries) |
| US-004.EC-3, INV-7 | Boot reconciles before advertising; disabled surface never returns | Integration | IT-005 | — |
| US-004 | Disable is immediate and unconditional | Domain unit | UT-019 | Integration IT-006 (a live remote session must actually stop being served) |
| US-004.EC-1 | Disable ends remote sessions, local unaffected | Integration | IT-006 | — |
| US-004.EC-2 | Disable mid-delivery: in-flight completes, new rejected | Domain unit | UT-020 | — |
| US-004.EC-4 | Double disable is a no-op result, not an error | Domain unit | UT-021 | — |
| US-001, US-001.EC-3/EC-4 | Status reports desired vs observed, transitions, and empty states truthfully | Domain unit | UT-022, UT-023, UT-024 | Integration IT-007 (HTTP/UDS/CLI report identical facts) |
| US-001.EC-1 | Degraded reachability is reported as degraded, never as stale-live | Domain unit | UT-025 | — |

### Tier listeners and route matrices

| Source | Invariant | Owning layer | IDs | Other layers (distinct failure) |
| --- | --- | --- | --- | --- |
| INV-1 | Daemon never binds a non-loopback address | Integration | IT-010 | — |
| INV-3, B-001 | Pairing mint/redeem routes exist on the private tier only and on no public configuration | Integration | IT-011, IT-012 | — |
| INV-3 | Public tier serves exactly its declared inventory; operator routes absent unless consented | Integration | IT-013 | — |
| INV-2 | Tier listeners grant no loopback trust | Integration | IT-014 | — |
| US-012, US-012.EC-3 | Consented public operator UI becomes reachable; withdrawal removes it while ingress persists; each re-enable needs fresh consent | Integration | IT-015 | E2E-005 (operator-visible consent flow) |
| US-012.EC-1 | An abandoned consent flow leaves the surface off | Domain unit | UT-027 | — |
| US-011.EC-3 | Public and private tiers enable independently | Domain unit | UT-026 | — |
| B-002 | Resolved tier ports are published before advertisement | Integration | IT-016 | — |
| US-013, US-013.EC-2 | Unauthenticated probing yields uniform responses, no enumeration, bounded rejection | Integration | IT-017 | — |
| US-013.EC-1 | Public-tier load does not starve private-tier/local handlers | Integration | IT-018 | — |

### Device credentials, pairing, revocation

| Source | Invariant | Owning layer | IDs | Other layers (distinct failure) |
| --- | --- | --- | --- | --- |
| US-005, N-001 | Credential is 256-bit random, stored only as a unique hash, compared constant-time | Domain unit | UT-030, UT-031 | — |
| US-005.EC-1/EC-2 | Expired and malformed artifacts are rejected without an existence oracle | Domain unit | UT-032, UT-033 | — |
| US-005.EC-3/EC-4, INV-10 | Redeem is atomic and single-use: exactly one winner | Domain unit | UT-034 | Integration IT-020 (two real HTTP clients race one artifact) |
| US-005.EC-5 | Pending artifacts are bounded; expiry frees slots | Domain unit | UT-035 | — |
| US-005.EC-6 | Local mint works with zero exposure; remote redeem needs a reachable tier | Domain unit | UT-036 | — |
| US-006, US-006.EC-4 | Device list shows origin/last-seen, ordered by activity; rename persists | Domain unit | UT-037, UT-038 | Integration IT-021 (HTTP and UDS render identically) |
| US-006.EC-3, US-008.EC-3 | Revoke is idempotent and reports `changed` distinctly | Domain unit | UT-039 | — |
| US-007, INV-12 | Revocation cancels every live stream for the device before returning | Domain unit | UT-040, UT-041 | Integration IT-022 (a real SSE connection must actually terminate) |
| US-007.EC-1, INV-13 | In-flight mutation revalidates epoch and fails before commit | Domain unit | UT-042 | Integration IT-023 (transaction actually rolls back; no partial write) |
| US-006.EC-1/EC-2 | Self-revoke and last-device revoke behave as specified; local access survives | Domain unit | UT-043, UT-044 | — |
| US-007.EC-2 | Failed-auth is rate-limited per source; local exempt | Domain unit | UT-045 | — |
| US-007.EC-3, US-012.EC-2 | A revoked device is unauthenticated everywhere; only a fresh artifact readmits | Domain unit | UT-046 | — |
| US-009, US-009.EC-4, INV-14 | Stream tickets are single-use, short-TTL, and re-acquired on every reconnect | Domain unit | UT-047, UT-048 | Integration IT-024 (browser-shaped SSE/WS handshake over a tier listener) |
| B-009 | Browser receives an httpOnly/Secure/SameSite cookie and never a raw credential | Integration | IT-025 | — |
| B-009 | CLI credential is stored client-locally at `0600` and deleted with the profile | Domain unit | UT-049 | — |
| US-008 | Agent device management is machine-readable with deterministic errors | Transport unit | UT-050 | Integration IT-021 |
| US-008.EC-1/EC-2 | Unknown id → deterministic not-found; empty list is a valid result | Domain unit | UT-051, UT-052 | — |

### Connectivity providers and verification

| Source | Invariant | Owning layer | IDs | Other layers (distinct failure) |
| --- | --- | --- | --- | --- |
| US-002, US-024 | Enable establishes, verifies, then advertises — in that order | Domain unit | UT-060 | Integration IT-030 (real subprocess provider through the extension seam) |
| US-002.EC-1/EC-2 | Abandoned or unauthorized enable leaves prior state, no partial exposure | Domain unit | UT-061, UT-062 | — |
| INV-8, B-008 | Challenge proves the endpoint reaches this daemon's assigned tier | Domain unit | UT-063, UT-064 | Integration IT-031 (endpoint pointed at the wrong tier must fail end to end) |
| B-008 | Verification enforces scheme allowlist, TLS validation, no-redirect, size bound, deadline, SSRF guard | Domain unit | UT-065–UT-070 | — |
| US-001.EC-2, US-024.EC-1 | Unverified endpoint is never advertised; provider marked unhealthy | Domain unit | UT-071 | — |
| INV-9, B-008 | Provider trust is re-derived from the live registry on enable and boot; mismatch fails closed | Domain unit | UT-072, UT-073 | Integration IT-032 (extension updated between boots) |
| US-025, US-025.EC-1 | Third-party provider requires digest confirmation; expanded declared control invalidates it and requires re-consent | Domain unit | UT-074, UT-075 | — |
| US-025.EC-2 | Provider acting outside its declared role is refused and audited | Domain unit | UT-076 | — |
| B-008 | Workspace-scoped provider sources are rejected for this surface | Domain unit | UT-077 | — |
| US-002.EC-3, US-024.EC-2 | Provider failure/outage degrades safely; teardown is bounded | Domain unit | UT-078, UT-079 | — |
| US-024.EC-3 | Two providers for one tier: operator selects; no double exposure | Domain unit | UT-080 | — |
| US-024 | Provider protocol conformance: negotiation, malformed payloads, crash, teardown, redaction | Protocol conformance | UT-081–UT-085 | — |

### Public ingress: webhooks and bridges

| Source | Invariant | Owning layer | IDs | Other layers (distinct failure) |
| --- | --- | --- | --- | --- |
| US-014, US-014.EC-1/EC-3 | Every trigger projects its public delivery URL; empty and pre-existing cases handled | Domain unit | UT-090, UT-091 | — |
| US-015 | Signed fresh delivery verifies and dispatches on the public tier | Integration | IT-040 | — |
| US-015.EC-1/EC-2/EC-4, US-016.EC-3 | Stale timestamp, oversized payload, disabled trigger, and a sender/trigger secret mismatch each reject distinctly with no side effect and an operator-diagnosable reason | Integration | IT-041 | — |
| US-015.EC-3 | Burst within bounds accepted; over-bound is retry-appropriate, never a silent drop | Integration | IT-042 | — |
| US-015.EC-5 | Two identical deliveries race → exactly one dispatch | Integration | IT-043 | — |
| US-016, US-016.EC-2 | A signed push starts a Loop attributed to trigger+delivery; refusal is recorded with reason | E2E | E2E-003 | Integration IT-044 (attribution recorded even when the E2E path is unavailable) |
| US-016.EC-1 | Daemon offline at delivery → sender-visible failure (documented durability boundary) | Integration | IT-045 | — |
| US-018, US-018.EC-1/EC-2 | Ingress verification is independent of operator auth; wrong-credential confusion is diagnosable operator-side only | Integration | IT-046 | — |
| US-017, US-017.EC-1/EC-3 | Bridge binds per instance; public-off marks broken ingress; external proxy coexists | Domain unit | UT-092, UT-093 | Integration IT-047 (platform callback reaches bridge verification) |
| US-011.EC-2, US-014.EC-2, US-017.EC-2, B-006 | Address change bumps `endpoint_generation`, invalidating confirmations and flagging dependents | Domain unit | UT-094 | Integration IT-048 (trigger URL displays and bound bridges both flagged) |
| B-006 | Ingress binding authorizes through the canonical workspace boundary; cross-workspace denied | Domain unit | UT-095, UT-096 | Integration IT-049 (agent caller from another workspace denied end to end) |
| B-006 | Binding is removed transactionally when its subject is deleted; orphans swept | Domain unit | UT-097 | Integration IT-050 (delete/recreate cycle) |
| US-011, US-011.EC-1 | Public address is stable across restarts; unstable address is refused | Integration | IT-051 | — |

### Remote clients: CLI profiles and SSH

| Source | Invariant | Owning layer | IDs | Other layers (distinct failure) |
| --- | --- | --- | --- | --- |
| US-019 | Pairing a CLI creates a profile plus a client-local credential and a daemon device row | Integration | IT-060 | — |
| US-019.EC-1/EC-3 | Unreachable target and duplicate profile name give distinct deterministic errors | Domain unit | UT-100, UT-101 | — |
| US-019.EC-2 | A copied encrypted profile export imports as the same device identity: visible and revocable | Integration | IT-061 | — |
| US-020 | Remote-capable verbs behave identically to local, including streaming | Integration | IT-062 | E2E-002 (operator-visible remote workflow) |
| B-010 | Local-only operations fail with `gateway_local_only_operation` against a remote profile | Transport unit | UT-102 | Integration IT-063 (agent-kernel/hosted-MCP/resource routes absent from tier listeners) |
| INV-17, B-010 | The gateway owns no queue/scheduler/claimer; no remote path can claim or heartbeat | Integration | IT-064 | — |
| US-020.EC-1/EC-2 | Unreachable and mid-stream loss produce distinct outcomes; output stays valid or explicitly truncated | Domain unit | UT-103, UT-104 | — |
| US-020.EC-4, US-009.EC-3 | Two clients on one daemon see consistent state | Integration | IT-065 | — |
| US-009.EC-2, US-021 | Remote-started work is detached and observable after reconnect; agent errors are stable | Integration | IT-066 | E2E-002 |
| US-021.EC-1/EC-2 | A revoked profile in a loop stays isolated; degraded output stays valid | Domain unit | UT-105 | — |
| US-022 | SSH connect launches or verifiably reuses the remote daemon and serves locally | Integration | IT-070 | E2E-004 |
| US-022.EC-1/EC-2/EC-5/EC-6 | Absent install, version mismatch, changed host key, non-default home each fail or adapt distinctly | Domain unit | UT-110–UT-113 | — |
| US-022.EC-3/EC-4 | Drop pauses locally while remote work continues; second connect reuses or reports busy | Integration | IT-071 | — |
| US-023, US-023.EC-1/EC-2 | SSH exposes nothing beyond loopback on either machine and never alters remote gateway state | Integration | IT-072 | — |
| US-009, US-009.EC-1, US-010, US-010.EC-1/EC-2 | Full product over the private tier for paired devices; reachable-unpaired sees only the gate; offline is a reachability error | Integration | IT-073 | E2E-001 |

### Cross-cutting: security, audit, registries, schema

| Source | Invariant | Owning layer | IDs | Other layers (distinct failure) |
| --- | --- | --- | --- | --- |
| INV-18, B-012 | No raw `compozy_claim_*` crosses HTTP, UDS, CLI, SSE, channel, event, or memory surfaces — only hash forms | Domain unit | UT-120 | Integration IT-080 (byte-scan across every live gateway surface) |
| US-027, INV-18 | Device credentials, pairing artifacts, and stream tickets appear only redacted | Domain unit | UT-121 | Integration IT-081 |
| US-027.EC-1/EC-2 | Secret-shaped values in wrong fields and provider subprocess output are redacted before logging | Domain unit | UT-122 | Protocol conformance UT-085 |
| US-026, US-026.EC-1/EC-2/EC-3 | Audit reports posture with stable ids and remediation; outage is a finding; empty is explicit; runs are side-effect-free | Domain unit | UT-130–UT-133 | E2E-006 (remediation clears a finding) |
| US-028, US-028.EC-1/EC-2 | Remote actions carry `actor_kind='operator_device'` / `actor_id`; local distinguishable; filterable | Domain unit | UT-140 | Integration IT-082 (events queryable by actor through existing paths) |
| B-011 | Every gateway event is registered with component and metadata; durable append precedes publish | Registry | UT-150, UT-151 | — |
| B-011 | Every gateway route is in the API operation registry with auth metadata; HTTP/UDS parity holds | Registry | UT-152 | Integration IT-083 |
| B-011 | `compozy__gateway` descriptor → binding → availability → generated catalog is complete and drift-free | Registry | UT-153, UT-154 | — |
| N-005, B-004 | Migration applies fresh, reopens idempotently, fails closed when ahead, and matches the declarative fragment | Schema | UT-160 | — |
| B-004 | Both-tier provider activation and dual-tier operator UI survive a reopen | Schema | UT-161 | — |
| N-006 | QA pair runs isolated and tears down with `teardown.json` `"clean": true` | E2E | E2E-007 | — |

## Unit Tests

### Config (`internal/config/gateway_test.go`, `internal/cli/config_test.go`)

- **UT-001** (happy): `defaultGatewayConfig` returns `enabled=false`, ports `0`, pairing `5m`/`8`, ticket `30s`, rate limit `10`/`60s`, verify `10s`, and public DNS-over-TLS resolver `1.1.1.1:853`.
- **UT-002** (error): `Validate` with `private_port=70000` → `ValidationError{Path:"gateway.private_port"}`.
- **UT-003** (error): `Validate` with `pairing.max_pending=-1` → `ValidationError` naming the path.
- **UT-004** (state): the settable-key map and overlay contain **no key that enables a surface** — asserts `gateway.public_ui.enabled` is absent, protecting INV-4 at the config layer where the domain test cannot see it.
- **UT-005** (boundary): `gatewayOverlay.Apply` — nil leaves the destination unchanged; set overwrites.
- **UT-006** (state): `SectionGateway` classifies ceiling/TTL/rate-limit as `Live` and port changes as `RestartRequired`.

### Policy and state machine (`internal/gateway/policy_test.go`)

- **UT-010** (state): `Transition` writes desired state and bumps `generation` in one transaction; config values never influence intent.
- **UT-011** (error): with `gateway.enabled=false`, every transition returns `ErrExposureRefused` naming the ceiling.
- **UT-012** (error): a remote-tier transition with the auth model inactive → `ErrExposureRefused` with cause and fix text.
- **UT-013** (ordering): enabling `(operator_ui, public)` before pairing is possible → refusal explaining the required order.
- **UT-014** (error): a removed legacy exposure semantic → refused, no compatibility interpretation.
- **UT-015** (concurrency): two transitions race; the loser's write is rejected on `generation`.
- **UT-016** (concurrency): an effect completing against a stale `generation` is discarded, not applied.
- **UT-017** (state): the reconciler applies effects in order persist→bind→establish→verify→advertise.
- **UT-018** (error): failure at each effect unwinds in reverse; no intermediate observed state is externally reachable.
- **UT-019** (happy): `Disable` marks desired `disabled` and observed `off` immediately.
- **UT-020** (concurrency): disable mid-delivery lets the in-flight request finish and rejects the next.
- **UT-021** (idempotency): `Disable` twice → `Changed=false`, no error.
- **UT-022** (happy): status projects tiers, surfaces, addresses, device count, provider health.
- **UT-023** (state): status distinguishes desired from observed during a transition.
- **UT-024** (boundary): zero devices/providers/bindings → explicit empty collections.
- **UT-025** (state): a degraded provider is reported degraded with cause; no stale address is shown live.
- **UT-026** (state): private and public tiers enable independently.
- **UT-027** (state): a public operator-UI consent flow abandoned before confirmation leaves the surface `disabled`; no partial consent is recorded.

### Device service (`internal/gateway/device_service_test.go`, `pairing_test.go`, `connection_registry_test.go`)

- **UT-030** (happy): a minted credential is 256-bit random with the `cpz_gwd_` prefix; the row stores only the SHA-256 hex under the unique index.
- **UT-031** (state): `Authenticate` compares constant-time and returns the session; a wrong credential of equal length takes the same comparison path.
- **UT-032** (error): expired artifact → `ErrPairingExpired`, no session created.
- **UT-033** (error): malformed artifact → `ErrPairingInvalid`; the response is identical whether or not an artifact exists.
- **UT-034** (concurrency): two redeemers race one artifact → exactly one session; the other gets `ErrPairingSpent`.
- **UT-035** (boundary): minting beyond `max_pending` → `ErrPairingLimit`; expiry frees slots.
- **UT-036** (state): local mint succeeds with zero exposure; redeem requires a reachable tier.
- **UT-037** (happy): device list carries name, origin, created/last-seen; rename persists.
- **UT-038** (state): list ordering is by last activity; long-inactive devices are selectable for bulk cleanup.
- **UT-039** (idempotency): revoke twice → second returns `Changed=false`; one canonical event.
- **UT-040** (state): `RevokeDevice` bumps `revoke_epoch` and calls `CancelDevice` before returning.
- **UT-041** (concurrency): `ConnectionRegistry` registers/deregisters concurrently without leaking handles; `CancelDevice` cancels exactly the target device's handles.
- **UT-042** (state): `RevalidateForCommit` with a stale epoch returns an error so the mutation aborts.
- **UT-043** (state): revoking the acting device succeeds and its next call is unauthenticated.
- **UT-044** (state): revoking the last remote device leaves local access working and re-pairing possible.
- **UT-045** (boundary): failed auth beyond `max_fails` within `window` is rate-limited; local is exempt.
- **UT-046** (error): a revoked credential is `ErrDeviceUnauthenticated` at every entry point, including redeem.
- **UT-047** (happy): a ticket is single-use with `stream_ticket.ttl`; second consumption → `ErrStreamTicketInvalid`.
- **UT-048** (state): a reconnect requires a freshly minted ticket; a cached ticket fails.
- **UT-049** (state): the CLI credential file is written `0600` and removed with the profile.
- **UT-051** (error): unknown device id → `ErrDeviceNotFound`.
- **UT-052** (boundary): empty device list is a valid empty result, not an error.

### Transport mapping (`internal/api/core/errors_test.go`, `gateway_handlers_test.go`)

- **UT-050** (happy): gateway domain types map to contract DTOs with every field populated; `RevokeResult.Changed` surfaces in the body.
- **UT-102** (error): a local-only operation against a remote target maps to `gateway_local_only_operation`.
- **UT-106** (happy): each sentinel maps to its documented status/code — `ErrPairingExpired→410`, `ErrPairingLimit→429`, `ErrDigestConfirmationRequired→428`, `ErrProviderTrustStale→409`, `ErrIngressForbidden→403`.
- **UT-107** (error): an unmapped error → 500 masked on HTTP, raw on UDS.

### Provider supervision and verification (`internal/gateway/provider_supervisor_test.go`, `endpoint_verify_test.go`)

- **UT-060** (happy): enable performs establish → verify → advertise in order.
- **UT-061** (state): abandoned authorization leaves prior state; retry starts clean.
- **UT-062** (error): provider reporting insufficient permission/quota fails enable with no partial exposure.
- **UT-063** (happy): the challenge nonce served on the assigned tier and echoed by the endpoint verifies.
- **UT-064** (error): an endpoint echoing the *other* tier's nonce, or no nonce, fails `ErrEndpointUnverified`.
- **UT-065** (error): a public-tier `http://` endpoint is rejected by the scheme allowlist.
- **UT-066** (error): an invalid TLS chain fails; `InsecureSkipVerify` is never used.
- **UT-067** (error): a redirect response fails verification (redirects are not followed).
- **UT-068** (boundary): a response body beyond 64 KiB is rejected without unbounded allocation.
- **UT-069** (boundary): verification honors `gateway.verify.timeout` on a dedicated client; `http.DefaultClient` is not used.
- **UT-070** (error): a public endpoint resolving to loopback/private/CGNAT fails the SSRF guard.
- **UT-071** (state): an unverified endpoint is never advertised; the provider is marked unhealthy with a redacted reason.
- **UT-072** (state): install source and digest are recomputed from the live registry on enable; the persisted row is not trusted.
- **UT-073** (error): a digest mismatch after an extension update → `ErrProviderTrustStale`, fail closed.
- **UT-074** (error): a third-party provider without confirmation → `ErrDigestConfirmationRequired`.
- **UT-075** (state): expanded declared control invalidates the prior confirmation and requires re-consent.
- **UT-076** (error): a provider acting outside its declared role is refused and an audit event is emitted.
- **UT-077** (error): a workspace-scoped provider source is rejected for this surface.
- **UT-078** (state): provider outage after enable → `degraded`, no silent fallback.
- **UT-079** (boundary): a hanging teardown is bounded; shutdown is not blocked.
- **UT-080** (concurrency): two providers desiring one tier → operator selection required; the partial unique index prevents two active.
- **UT-162** (boundary): public verification uses the configured public DNS resolver instead of the host resolver.
- **UT-163** (state): a failed endpoint proof retains the unverified provider session and challenge for a later proof without advertisement.
- **UT-164** (state): an initial endpoint-proof failure schedules recovery, keeps the tier fail-closed, and advertises only after a later proof succeeds.
- **UT-165** (integration boundary): the bundled Tailscale provider provisions the HTTPS certificate for the selected domain before opening its public Funnel listener.
- **UT-166** (boundary): provider activation receives its own bounded deadline instead of consuming the shorter endpoint-proof timeout.
- **UT-167** (security): public DNS resolution authenticates its DNS-over-TLS peer and rejects an invalid certificate.

### Provider protocol conformance (`internal/extension/connectivity_conformance_test.go`, real subprocess)

- **UT-081** (happy): initialize negotiates `connectivity.provider` and its service methods; only implemented methods are callable.
- **UT-082** (error): a malformed JSON-RPC payload from the provider is rejected without crashing the daemon.
- **UT-083** (error): calling a method the provider did not implement fails deterministically.
- **UT-084** (state): provider crash and restart are supervised; teardown after crash is bounded.
- **UT-085** (state): provider stdout containing credential-shaped material is redacted before logging.

### Ingress projection and bindings (`internal/gateway/ingress_test.go`)

- **UT-090** (happy): a trigger's public delivery URL is projected from the endpoint plus `FormatWebhookEndpoint`; pre-existing triggers gain it without re-creation.
- **UT-091** (boundary): with public ingress off, the URL surface reports off and offers the enable path instead of a dead URL; zero triggers is an explicit empty state.
- **UT-092** (happy): a bridge binds per instance with confirmation; binding is recorded with `endpoint_generation`.
- **UT-093** (state): public-off marks bound bridges broken-ingress; an external-proxy bridge coexists untouched.
- **UT-094** (state): an address change bumps `endpoint_generation`, invalidating confirmations and flagging dependents.
- **UT-095** (error): a bind request carrying a client-supplied workspace is ignored; the owner is resolved server-side.
- **UT-096** (error): a caller from another workspace → `ErrIngressForbidden`.
- **UT-097** (state): deleting a subject removes its binding in the same transaction; an orphan is swept on reconcile.

### CLI transport and profiles (`internal/cli/gateway_client_test.go`)

- **UT-100** (error): an unreachable target gives a reachability error distinct from an auth error.
- **UT-101** (error): a duplicate profile name → deterministic conflict with rename/overwrite choice.
- **UT-103** (error): mid-stream loss reports interruption with a distinct exit outcome.
- **UT-104** (state): structured output during degradation stays valid or reports explicit truncation.
- **UT-105** (idempotency): a revoked profile in a loop yields a stable auth error; other profiles are unaffected.
- **UT-108** (happy): the client builds `https://host:port` with the bearer header for a remote profile and `http://unix` for the socket, at the single chokepoint.

### SSH method (`internal/cli/connect_ssh_test.go`)

- **UT-110** (error): Compozy absent on the remote → deterministic install-required error, no partial bootstrap.
- **UT-111** (error): version incompatibility → refusal naming both versions.
- **UT-112** (error): a changed host key → refusal with the standard warning; never auto-accepted.
- **UT-113** (state): a non-default remote home is honored and recorded in the profile.

### Security, audit, attribution (`internal/gateway/audit_test.go`, `internal/api/core/gateway_handlers_test.go`)

- **UT-120** (state): no raw `compozy_claim_*` value appears in any gateway-produced payload, log line, event, or status field; only hash forms.
- **UT-121** (state): device credentials, pairing artifacts, and stream tickets appear only as truncated/hashed forms.
- **UT-122** (state): a secret-shaped value placed in a wrong field is redacted before logging or rendering.
- **UT-130** (happy): audit reports tiers, addresses, per-surface reachability, auth posture, device highlights, provider health, ingress posture, ranked with remediation.
- **UT-131** (state): a provider outage is a finding, not an audit failure.
- **UT-132** (boundary): no findings → explicit no-findings result, distinct from "did not run".
- **UT-133** (idempotency): repeated runs are side-effect-free and stable absent state change.
- **UT-140** (happy): a remote action records `actor_kind='operator_device'` and `actor_id=<device id>`; a local action records its existing actor values.

### Registries (`internal/events/registry_test.go`, `internal/api/spec`, `internal/tools/builtin/builtin_test.go`)

- **UT-150** (state): every `gateway.*` event name is registered with `ComponentGateway` and its severity/global metadata.
- **UT-151** (ordering): each gateway event is durably appended before it is published to live consumers.
- **UT-152** (state): every gateway route appears in the API operation registry with auth metadata; HTTP and UDS registrations agree.
- **UT-153** (state): the `compozy__gateway` descriptor resolves through binding, dependencies, and availability diagnostics.
- **UT-154** (state): the generated native tool catalog matches the descriptor set with no drift.

### Schema (`internal/store/globaldb` migration suites)

- **UT-160** (state): the appended migration applies to a fresh DB, reopens idempotently, fails closed when the DB is ahead, and matches the declarative fragment.
- **UT-161** (state): a provider activated on both tiers and an operator UI enabled on both tiers survive a reopen with keys intact; the partial unique index rejects a second active provider per tier.

## Integration Tests

### Exposure and listeners

- **IT-001**: with the auth model inactive, a tier listener either does not open or serves nothing; the local listener and UDS are unaffected.
- **IT-002**: booting with a hand-edited forbidden `[gateway]` combination starts local-only, reports the refused config with its fix, and does not crash-loop.
- **IT-003**: concurrent transitions issued over HTTP and UDS resolve to one applied state.
- **IT-004**: fault injection at each of persist/bind/establish/verify/advertise leaves no externally reachable intermediate; compensation unwinds in reverse.
- **IT-005**: kill the daemon mid-transition, restart → reconcile completes before advertisement; a disabled surface stays disabled.
- **IT-006**: disabling a tier terminates its live sessions while local access continues.
- **IT-007**: gateway status is identical through HTTP, UDS, and `compozy daemon status`.
- **IT-010**: the process binds no non-loopback socket in any exposure configuration.
- **IT-011**: `POST /api/gateway/pairings/redeem` is routable on the private tier listener.
- **IT-012**: the same path is **absent** from the public tier listener in every configuration, including with the operator UI consented; pairing mint is likewise absent.
- **IT-013**: the public listener serves exactly its declared inventory; operator routes appear only after consent and disappear on withdrawal.
- **IT-014**: a request arriving on a tier listener from loopback receives no implicit trust.
- **IT-015**: consenting to the public operator UI makes it reachable for a paired device; withdrawing removes it while webhook ingress keeps working.
- **IT-016**: resolved tier ports are published into status before any endpoint is advertised.
- **IT-017**: unauthenticated probing of arbitrary paths yields uniform responses with no enumeration; malformed and oversized requests are bounded.
- **IT-018**: sustained public-tier load leaves private-tier and local handlers responsive.

### Devices and streams

- **IT-020**: two real HTTP clients race one pairing artifact → exactly one device session.
- **IT-021**: device list/rename/revoke render and behave identically over HTTP and UDS.
- **IT-022**: revoking a device with an open SSE connection terminates that connection before the revoke call returns.
- **IT-023**: revoking during an in-flight mutation aborts the transaction with no partial write.
- **IT-024**: an SSE and a WebSocket each connect with a fresh single-use ticket and are rejected when a ticket is reused.
- **IT-025**: browser redemption sets an httpOnly/Secure/SameSite cookie, and no raw credential appears in any response body or URL.

### Providers

- **IT-030**: a real subprocess provider enables through the extension seam, reports endpoints, and is advertised only after verification.
- **IT-031**: an endpoint pointed at the wrong tier fails verification end to end and is never advertised.
- **IT-032**: updating the provider extension between boots invalidates the confirmed digest and fails closed until re-consent.

### Ingress

- **IT-040**: a signed fresh delivery on the public tier verifies and dispatches.
- **IT-041**: stale timestamp, oversized payload, and disabled trigger each reject with their distinct code and no side effect.
- **IT-042**: a burst within bounds is accepted; over-bound deliveries get a retry-appropriate response.
- **IT-043**: two identical deliveries race → exactly one dispatch.
- **IT-044**: a dispatched delivery records trigger and delivery attribution on the resulting run.
- **IT-045**: with the daemon stopped, the sender observes a failed delivery (documented durability boundary).
- **IT-046**: an operator session posting unsigned to a delivery URL is rejected; a GET is method-rejected with no daemon detail.
- **IT-047**: a platform callback to a bound bridge reaches that bridge's verification path.
- **IT-048**: an address change flags trigger URL displays and bound bridges for re-confirmation.
- **IT-049**: an agent caller from another workspace is denied binding/unbinding a foreign subject.
- **IT-050**: deleting and recreating a subject leaves no orphaned binding and requires fresh confirmation.
- **IT-051**: the public address survives a daemon restart unchanged; a provider unable to guarantee stability fails enable.

### Remote clients

- **IT-060**: pairing a CLI creates the profile, the client-local credential, and the daemon device row together.
- **IT-061**: a copied passphrase-protected profile export imports on a second client as the same device and is revocable from either client.
- **IT-062**: remote-capable verbs behave identically to local, including streaming output.
- **IT-063**: agent-kernel, hosted-MCP, MCP Host API, and resource-mutation routes are absent from both tier listeners.
- **IT-064**: no remote path can claim, heartbeat, complete, fail, or release a task run; `ClaimNextRun` authority is unreachable remotely.
- **IT-065**: two clients operating one remote daemon observe consistent state.
- **IT-066**: work started remotely survives client disconnect and is observable on reconnect.
- **IT-070**: SSH connect launches (or verifiably reuses) the remote daemon and serves it locally; teardown stops only what it started.
- **IT-071**: an SSH drop pauses local access while remote work continues; a second connect reuses or reports busy.
- **IT-072**: no new surface is reachable beyond loopback on either machine, and the remote's own gateway state is untouched.
- **IT-073**: a paired device reaches the full product over the private tier; a reachable-unpaired client sees only the gate; a stopped daemon yields a reachability error.

### Cross-cutting

- **IT-080**: a byte-scan of every live gateway surface (HTTP tiers, UDS, CLI output, SSE frames, events) finds no raw `compozy_claim_*` value.
- **IT-081**: the same scan finds no raw device credential, pairing artifact, or stream ticket.
- **IT-082**: gateway events are queryable by `actor_id` through the existing observability paths.
- **IT-083**: HTTP and UDS gateway operations match their API-registry entries, including auth metadata.

## End-to-End Tests

- **E2E-001** *(US-002, US-005, US-009, US-010)* — E2E web: enable the private tier with the fixture provider → mint a pairing → a second browser context redeems it → the full UI loads with a live stream; an unpaired context sees only the gate.
- **E2E-002** *(US-019, US-020, US-021, US-009.EC-2)* — E2E runtime: pair a CLI profile → run a remote command that starts long work → disconnect and reconnect → observe progress; a local-only verb fails with its deterministic error.
- **E2E-003** *(US-014, US-015, US-016)* — E2E runtime: enable public ingress → a trigger bound to a Loop shows its delivery URL → post a signed delivery → a Loop run starts attributed to trigger and delivery.
- **E2E-004** *(US-022, US-023)* — E2E runtime: `compozy connect ssh <host>` against the harness remote → daemon launched or reused → local access works → disconnect leaves no new reachable surface.
- **E2E-005** *(US-012, US-004, US-007)* — E2E web: consent to the public operator UI → reach it from a paired device → revoke that device and watch its open view land on "access ended" → disable exposure and confirm local-only.
- **E2E-006** *(US-026)* — E2E web: run the self-audit → it reports a finding → apply the named remediation → re-run reports it cleared.
- **E2E-007** *(N-006)* — QA execution: the trailing QA pair runs in an isolated lab (unique home/ports/socket, derived web proxy target), walks every scenario added or reset by this feature to a recorded verdict, and ends with `teardown.json` `"clean": true`.
