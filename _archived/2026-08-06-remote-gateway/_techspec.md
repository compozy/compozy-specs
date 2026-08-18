# TechSpec: Remote Gateway

> Revision 2 — incorporates peer-review round 1 (`qa/peer-review-findings-round1.md`, verdict BLOCKED): all 12 blockers and 6 nits resolved. See `qa/peer-review-incorporation-round1.md` for the mapping.

## Executive Summary

The Remote Gateway makes a Compozy daemon reachable from outside its machine under a fail-closed, single-operator authentication model, without Compozy operating any hosted infrastructure. The daemon stays **loopback-only**: connectivity provider extensions terminate external transport and forward to daemon-owned loopback **tier listeners** — a private tier carrying the operator surface behind device authentication, and a public tier carrying webhook/bridge ingress by default with the operator UI reachable only behind explicit consent. Authentication is an **opaque, revocable device-session credential** (bearer for the CLI, an httpOnly cookie for the browser) verified by a device-auth middleware, with **single-use stream tickets** for SSE/WebSocket and a **device-keyed connection registry** so revocation terminates live streams. A new `connectivity.provider` extension provide surface — public, open to third parties under existing install-source tiers plus digest confirmation re-derived from the live extension registry — carries the mechanism; a bundled first-party provider embeds Tailscale via `tsnet`.

Exposure is a **durable desired/observed state machine** reconciled at boot and on every transition, with the runtime intent owned exclusively by the database and configuration holding only static tunables and an immutable ceiling. Listener construction is owned by `internal/daemon` (the composition root that already owns server factories); `internal/gateway` owns policy and computes the desired exposure plan. Remote client reach is bounded by an explicit **operation matrix**: the operator/management surface is remote-capable, while the agent kernel, hosted MCP, and Host API stay UDS-only — the gateway owns no queue, scheduler, or claim primitive.

Primary trade-offs (ADRs 007–011): topology keeps TLS and reachability in the provider and makes fail-closed structural (tier-by-listener, never tier-by-header); the opaque-credential model is chosen over proof-of-possession with revocation plus a connection registry as containment; state persists in the existing Global DB and vault rather than a new database; the subsystem is one cohesive `internal/gateway` domain package.

## MVP Boundary

**MVP boundary:** every component in this TechSpec is MVP — config/home/boundaries (Development Sequencing step 1), gateway policy + desired/observed state machine + reconciler (step 2), tier listeners owned by the daemon (step 3), device sessions + pairing + connection registry (step 4), device-auth middleware + stream tickets + browser credential flow (step 5), the `connectivity.provider` surface + verification protocol + provider supervision (step 6), the bundled `tsnet` provider (step 7), webhook/bridge public reachability (step 8), remote CLI profiles within the operation matrix (step 9), the SSH connection method (step 10), generated/native-tool/event surfaces (step 11), self-audit + web + docs (step 12), and the QA planning/execution pair (step 13). **Post-MVP (out of scope, per PRD Non-Goals):** any Compozy-hosted relay/cloud, multi-user accounts/roles, native mobile apps, cross-machine Compozy Network peering, webhook store-and-forward durability, additional bundled providers beyond Tailscale, and proof-of-possession device credentials (deferred per ADR-008 — no speculative schema now).

## System Architecture

### Component Overview

Domain logic lives in `internal/gateway`, composed by `internal/daemon`; transport handlers are shared `BaseHandlers` methods in `internal/api/core`; the bundled provider is a repo-root `extensions/connectivity/tailscale/` package.

- **`gateway.Policy`** — the single authority for exposure intent. Owns the durable desired/observed state machine, serializes transitions, enforces fail-closed refusals, and computes the **exposure plan** (which tiers must be up and which surface set each carries). It does not bind sockets.
- **`gateway.Reconciler`** — drives observed state toward desired state at boot and after every transition, in a fixed effect order with compensation. Advertises endpoints only after verification succeeds.
- **`gateway.DeviceService`** — pairing-artifact mint/redeem (in-memory, bounded), device-session lifecycle, constant-time credential verification, single-use stream tickets, and the revocation epoch.
- **`gateway.ConnectionRegistry`** — device-keyed registry of live stream cancel handles. Every authenticated stream registers on connect and deregisters on close; revocation cancels all handles for a device before returning.
- **`gateway.ProviderSupervisor`** — resolves the active connectivity provider per tier through the extension consumer, drives establish/status/teardown, runs the **endpoint verification protocol**, re-derives provider trust from the live extension registry, and supervises health with fail-safe degradation.
- **`gateway.Auditor`** / **`gateway.StatusProjector`** — findings computation and the status view.
- **`internal/daemon` tier-listener owner** — constructs, starts, publishes the resolved ports of, and shuts down the tier listeners using the existing server-factory pattern. This is the composition root's job, not the domain package's (B-002).
- **`core.GatewayService`** (interface over `internal/gateway` domain types) + `BaseHandlers` gateway methods + `internal/api/contract` wire DTOs — the transport facade.
- **`extension` connectivity consumer** — a `ConnectivitySource` resolver mirroring `internal/extension/watch_source.go`.
- **`extensions/connectivity/tailscale/`** — bundled provider: a `tsnet` node establishing the tailnet listener and the Funnel listener, forwarding each to the assigned loopback tier listener.
- **CLI** — a target-aware client transport (unix socket vs https+bearer), client-local credential storage, `[[gateway.connections]]` profiles, and the `gateway` / `pair` / `device` / `connect` command groups.

### Data Flow

- **Operator, private tier:** device → provider (tailnet, TLS) → private tier listener → device-auth middleware → `BaseHandlers` → domain services; streams authenticate with a single-use ticket and register in the connection registry.
- **External sender, public tier:** GitHub → provider (Funnel, TLS) → public tier listener → webhook route (registered ahead of device auth) → existing HMAC/freshness/replay verification → automation dispatch → Loop start.
- **Pairing:** a trusted surface (local UDS/CLI, or an authenticated private-tier device) mints an artifact → the new device redeems it **on the private tier only** → receives a credential → appears in the device list. The public tier has no mint and no redeem route (B-001).
- **Provider lifecycle:** `Policy` records desired state → `Reconciler` starts the provider → provider reports endpoints → supervisor verifies each endpoint with the challenge protocol → observed state advances → status advertises.

## Architectural Boundaries

- **`internal/gateway`** is a leaf domain package. It MUST NOT import `internal/daemon`, `internal/api/contract`, `internal/api/core`, `internal/api/httpapi`, `internal/api/udsapi`, or `internal/cli`. It receives collaborators (store, vault, provider resolver, clock, listener-port publisher) via constructor functional options.
- **Type ownership (B-003):** `internal/gateway` owns the domain request/result types (`ExposureRequest`, `PairingArtifact`, `DeviceSession`, `RevokeResult`, `Reachability`, `AuditReport`, …). `internal/api/core` defines the `GatewayService` interface **over those gateway types** (the existing pattern — `core` imports domain packages; domain packages never import `core`). Wire DTOs live in `internal/api/contract`, and the `BaseHandlers` gateway methods perform explicit gateway-type → contract-DTO mapping. No aliasing, no inverted dependency.
- **Listener ownership (B-002):** `internal/gateway` computes the exposure plan; `internal/daemon` owns tier-listener construction via a `gatewayServerFactory` mirroring the existing `httpFactory`/`udsFactory`. Each tier listener is an `httpapi.Server` built with a tier-specific option set (`httpapi.WithSurfaceSet(tier)`, `httpapi.WithDeviceAuth(...)`) that selects the tier's route matrix. `EffectDriver.Bind` returns the resolved loopback address, and the reconciler records it in status before establishing or advertising the provider.
- **`internal/api/httpapi`** gains `WithDeviceAuth` and `WithSurfaceSet` options plus the tier route matrices; **`internal/api/udsapi`** registers the local/agent gateway management subset. Neither ever mounts the UDS-only superset (agent kernel, hosted MCP, Host API) on a tier listener.
- **`internal/extension`** gains the connectivity consumer and the `connectivity.provider` surface in `internal/extensionprotocol`. `internal/gateway` defines a small `ConnectivitySource` interface satisfied by an adapter in `internal/extension`; no back-pointer.
- **`extensions/connectivity/tailscale/`** lives outside `internal/`, embedded via `//go:embed` and re-exec'd as an extension provider subprocess.
- **`magefiles/boundaries.go`** gains the six `internal/gateway` → shell forbidden-import pairs (mirroring the `internal/marketplace` block) in the same change; `magefiles/magefile_test.go` updated alongside.
- **File split decided up front** (500-line cap): `types.go`, `policy.go`, `policy_transition.go`, `reconciler.go`, `device_service.go`, `device_credential.go`, `pairing.go`, `connection_registry.go`, `provider_supervisor.go`, `endpoint_verify.go`, `audit.go`, `status.go`, `store.go`.

## Implementation Design

### Core Interfaces

Transport facade (`internal/api/core`, over `internal/gateway` types; implemented by `internal/gateway`):

```go
type GatewayService interface {
    Status(ctx context.Context) (gateway.Status, error)
    SetSurfaceExposure(ctx context.Context, req gateway.SurfaceExposureRequest) (gateway.Status, error)
    EnableProvider(ctx context.Context, req gateway.ProviderEnableRequest) (gateway.Status, error)
    DisableProvider(ctx context.Context, tier gateway.Tier, providerName string) (gateway.Status, error)
    MintPairing(ctx context.Context, req gateway.PairingRequest) (gateway.PairingArtifact, error)
    RedeemPairing(ctx context.Context, req gateway.RedeemRequest) (gateway.IssuedCredential, error)
    ListDevices(ctx context.Context) ([]gateway.DeviceSession, error)
    RevokeDevice(ctx context.Context, deviceID string) (gateway.RevokeResult, error)
    RenameDevice(ctx context.Context, deviceID, name string) (gateway.DeviceSession, error)
    MintStreamTicket(ctx context.Context, deviceID string) (gateway.StreamTicket, error)
    BindIngress(ctx context.Context, req gateway.IngressBindRequest) (gateway.IngressBinding, error)
    UnbindIngress(ctx context.Context, ref gateway.IngressSubjectRef) (gateway.UnbindResult, error)
    Audit(ctx context.Context) (gateway.AuditReport, error)
}
```

`RevokeResult`/`UnbindResult` carry `Changed bool` so idempotent no-ops are distinguishable from state changes (B-003).

Device authentication and live-connection control (`internal/gateway`):

```go
type DeviceAuthenticator interface {
    Authenticate(ctx context.Context, credential string) (DeviceSession, error)
    ConsumeStreamTicket(ctx context.Context, ticket string) (DeviceSession, error)
    RevalidateForCommit(ctx context.Context, deviceID string, epoch uint64) error
}

type ConnectionRegistry interface {
    Register(deviceID string, cancel context.CancelFunc) (handle uint64)
    RegisterWaitable(deviceID string, cancel context.CancelFunc, done <-chan struct{}) (handle uint64)
    Deregister(deviceID string, handle uint64)
    CancelDevice(ctx context.Context, deviceID string) (canceled int, err error)
}
```

`Authenticate` is constant-time and returns `ErrDeviceUnauthenticated` for unknown, revoked, or mismatched credentials. `RevalidateForCommit` compares the caller's revocation epoch inside the mutation transaction so an in-flight state-changing call fails before commit if the device was revoked mid-flight (B-007).

Exposure state machine (`internal/gateway`):

```go
type Policy interface {
    Plan(ctx context.Context) (ExposurePlan, error)
    Transition(ctx context.Context, req TransitionRequest) (Status, error)
    Reconcile(ctx context.Context) (Status, error)
}
```

`Transition` writes desired state and bumps `generation` in one SQLite transaction, then hands the plan to the reconciler; `Reconcile` runs at boot before any advertisement and after any observed drift.

Connectivity provider consumer (`internal/gateway`, adapted by `internal/extension`; daemon→provider methods `connectivity/establish|status|teardown` ride the existing JSON-RPC stdio transport):

```go
type ConnectivitySource interface {
    Establish(ctx context.Context, req EstablishRequest) (Reachability, error)
    Status(ctx context.Context) (Reachability, error)
    Teardown(ctx context.Context) error
}

type EstablishRequest struct {
    Tier          Tier          // private | public
    ForwardTarget netip.AddrPort // the daemon's loopback tier listener
    ChallengePath string         // well-known path the daemon will probe
    Deadline      time.Time
}

type Reachability struct {
    Tier      Tier
    Endpoints []AdvertisedEndpoint // {URL, Scheme, Stability} — verified before advertisement
    Health    ProviderHealth       // healthy | degraded(reason)
}
```

Provider supervision and verification (`internal/gateway`):

```go
type ProviderSupervisor interface {
    Enable(ctx context.Context, req ProviderEnableRequest) error
    Disable(ctx context.Context, tier Tier) error
    Verify(ctx context.Context, tier Tier, ep AdvertisedEndpoint) error
}
```

Errors are sentinels wrapped with `%w` and matched with `errors.Is`: `ErrDeviceUnauthenticated`, `ErrDeviceNotFound`, `ErrStreamTicketInvalid`, `ErrPairingInvalid`, `ErrPairingExpired`, `ErrPairingSpent`, `ErrPairingLimit`, `ErrExposureRefused`, `ErrConsentRequired`, `ErrEndpointUnverified`, `ErrProviderDegraded`, `ErrProviderTrustStale`, `ErrDigestConfirmationRequired`, `ErrIngressSubjectNotFound`, `ErrIngressForbidden`, `ErrLocalOnlyOperation`. `internal/api/core/errors.go` gains `StatusForGatewayError` mapping each to a stable HTTP status + code.

### Endpoint Verification Protocol (B-008)

Verification proves an advertised endpoint actually reaches **this daemon's assigned tier listener** — a reachability probe alone cannot:

1. The daemon generates a per-attempt nonce and serves it at `ChallengePath` on the tier listener only (never on the local listener, never on the other tier).
2. The daemon fetches `<endpoint>/<ChallengePath>` and requires the exact nonce in the response body.
3. **Scheme allowlist:** public-tier endpoints MUST be `https`; private-tier endpoints MUST be `https` unless the provider declares a tailnet-internal scheme explicitly allowed by policy.
4. **TLS validation is mandatory** — no `InsecureSkipVerify`, ever.
5. **Redirects are not followed**; a redirect response fails verification.
6. **Response body is bounded** (64 KiB) and the whole request carries an explicit deadline (default 10s) on a dedicated client with a timeout — `http.DefaultClient` is forbidden.
7. **SSRF guard:** the resolved address must not be a loopback/link-local/private/CGNAT address for a public-tier endpoint (a public endpoint resolving inward means the provider is misconfigured or hostile).
8. **Public DNS isolation:** public-tier verification resolves through authenticated DNS-over-TLS at `gateway.verify.public_dns_resolver` (default `1.1.1.1:853`), never through host MagicDNS that may return the provider's private tailnet address.
9. Failure → `ErrEndpointUnverified`; the endpoint is never advertised and the provider is marked degraded with a redacted reason. The staged listener and challenge remain alive for bounded-backoff verification retries so providers with eventually consistent public DNS are not dismantled before publication. Disable, generation change, or shutdown tears the staged attempt down.

**Provider trust freshness (B-008):** `install_source` and the control digest are **re-derived from the live extension registry on every enable and every boot**, never trusted from the persisted row. A mismatch between the persisted `digest_confirmed` and the freshly computed digest fails closed with `ErrProviderTrustStale` and requires re-consent. Workspace-scoped extension sources are **rejected** for this surface: connectivity is an operator-global capability and a workspace-scoped source has no owner rule that could authorize it.

### Credential Ownership and Stream Flow (B-009, N-001)

- **Credential format (N-001):** 256 bits from `crypto/rand`, rendered as an opaque string with a `cpz_gwd_` prefix and a distinct `cpz_gwp_` prefix for pairing artifacts (so redaction and misuse diagnostics can tell them apart). The daemon stores only `sha256(credential)` hex in `token_hash`, with a `UNIQUE` index; verification looks the row up by hash and then runs `subtle.ConstantTimeCompare` on the hash bytes. Rotation issues a new credential and revokes the old session row.
- **Server side** stores hashes only — never the credential.
- **CLI client side** stores the credential in a **client-local** encrypted file under that machine's `$COMPOZY_HOME/gateway/credentials/<profile>.cred` (mode `0600`), because a remote CLI machine has no access to the daemon's vault. The `[[gateway.connections]]` entry holds only non-secret metadata plus the credential file reference; `compozy connect remove` deletes both. Cross-machine transfer uses `compozy connect export|import`: the export is encrypted with scrypt + AES-GCM from a passphrase supplied through a private file, and importing recreates the same credential/device identity without exposing it in command output. Copying metadata alone never transfers identity.
- **Browser side** never handles a raw credential: redeeming a pairing on the private tier sets an **httpOnly, Secure, SameSite=Lax** cookie scoped to the gateway origin, and the API accepts either that cookie (browser) or a bearer header (CLI). Cross-origin abuse is covered by the existing origin protection plus SameSite; responses carrying credentials set `Cache-Control: no-store` and `Referrer-Policy: no-referrer`.
- **Stream flow:** one ticket per stream — the client POSTs to `/api/gateway/stream-tickets` (authenticated), receives a single-use ticket with `stream_ticket.ttl`, and opens the SSE/WebSocket with it as a query parameter. The ticket is consumed at connect; **every reconnect acquires a fresh ticket**. The WebSocket upgrade uses the same ticket mechanism. On connect the stream registers in the `ConnectionRegistry`; on close it deregisters. Existing SSE/WebSocket consumers in `web/` are migrated to acquire a ticket when the active connection is a remote gateway session.

### Remote Operation Matrix (B-010)

Remote reach is bounded, published, and enforced — not "every verb":

| Surface | Local (UDS) | Private tier (remote) | Public tier |
| --- | --- | --- | --- |
| Operator/management API (sessions, tasks read, loops, memory, settings, bridges, extensions, gateway) | yes | **yes** | only when the operator UI surface is consented |
| Gateway management (status/exposure/providers/pairing/devices/audit) | yes | yes (pairing mint requires a trusted surface) | no |
| Webhook + bridge ingress | no (HTTP-only today) | no | **yes** (own HMAC verification) |
| Agent kernel (`/api/agent/*`: spawn, claim-next, heartbeat, complete, fail, release) | **yes, UDS-only** | no | no |
| Hosted MCP + MCP Host API (`/api/internal/*`) | **yes, UDS-only** | no | no |
| Resource mutation (`/api/resources/*`) | yes, UDS-only | no | no |

The CLI publishes the same matrix: verbs bound to UDS-only operations return `ErrLocalOnlyOperation` deterministically when a remote profile is active. **The gateway owns no queue, no scheduler, and no claim primitive**: task claim/heartbeat/complete/fail/release remain exclusively `task.Service` primitives reached over UDS on the daemon machine, preserving `ClaimNextRun` authority and claim-token fencing.

### Data Models

New Global DB tables (`internal/store/globaldb/schema/definitions/NN_gateway.sql`), with an appended Goose migration. Keys corrected for the tier topology (B-004) and the desired/observed model (B-005):

```sql
CREATE TABLE gateway_device_sessions (
  id             TEXT PRIMARY KEY,   -- opaque session id (dev_*)
  name           TEXT NOT NULL,      -- operator-facing device name
  token_hash     TEXT NOT NULL,      -- sha256 hex of the credential; raw never stored
  actor_kind     TEXT NOT NULL CHECK (actor_kind IN ('operator_device','cli_profile')),
  pairing_origin TEXT NOT NULL,      -- trusted surface that minted the pairing
  revoke_epoch   INTEGER NOT NULL DEFAULT 0, -- bumped on revoke; compared before mutation commit
  created_at     TIMESTAMP NOT NULL,
  last_seen_at   TIMESTAMP,          -- last authenticated request; null until first use
  revoked_at     TIMESTAMP           -- null = active; set = terminal (row retained for audit)
);
CREATE UNIQUE INDEX gateway_device_sessions_token_hash ON gateway_device_sessions(token_hash);

-- provider identity/enrollment, one row per installed connectivity provider
CREATE TABLE gateway_providers (
  name             TEXT PRIMARY KEY, -- installed extension name
  install_source   TEXT NOT NULL,    -- re-derived from the extension registry on enable/boot
  digest_confirmed TEXT,             -- operator-confirmed control digest; null for bundled first-party
  confirmed_at     TIMESTAMP
);

-- per-tier activation: one provider may serve both tiers; one active provider per tier
CREATE TABLE gateway_provider_activations (
  provider_name  TEXT NOT NULL REFERENCES gateway_providers(name) ON DELETE CASCADE,
  tier           TEXT NOT NULL CHECK (tier IN ('private','public')),
  desired_state  TEXT NOT NULL CHECK (desired_state IN ('enabled','disabled')),
  observed_state TEXT NOT NULL CHECK (observed_state IN ('down','establishing','up','degraded')),
  generation     INTEGER NOT NULL DEFAULT 0, -- bumped per desired-state write; reconciler fences on it
  last_health_at TIMESTAMP,
  last_error     TEXT,               -- redacted provider failure reason
  PRIMARY KEY (provider_name, tier)
);
CREATE UNIQUE INDEX gateway_active_provider_per_tier
  ON gateway_provider_activations(tier) WHERE desired_state = 'enabled';

-- surface reachability per tier: operator_ui is always private, optionally public
CREATE TABLE gateway_surface_exposure (
  surface        TEXT NOT NULL CHECK (surface IN ('operator_ui','webhook_ingress')),
  tier           TEXT NOT NULL CHECK (tier IN ('private','public')),
  desired_state  TEXT NOT NULL CHECK (desired_state IN ('enabled','disabled')),
  observed_state TEXT NOT NULL CHECK (observed_state IN ('off','on')),
  generation     INTEGER NOT NULL DEFAULT 0,
  consented_at   TIMESTAMP,          -- required for (operator_ui, public); re-set on each enable
  PRIMARY KEY (surface, tier)
);

-- ingress bindings carry the subject's owning scope for authorization (B-006)
CREATE TABLE gateway_ingress_bindings (
  subject_kind        TEXT NOT NULL CHECK (subject_kind IN ('webhook_trigger','bridge_instance')),
  subject_id          TEXT NOT NULL,
  scope_kind          TEXT NOT NULL CHECK (scope_kind IN ('global','workspace')),
  workspace_id        TEXT,          -- non-null iff scope_kind='workspace'; resolved, never client-supplied
  endpoint_generation INTEGER NOT NULL, -- provider/tier endpoint generation this confirmation is bound to
  confirmed_at        TIMESTAMP NOT NULL,
  PRIMARY KEY (subject_kind, subject_id),
  CHECK ((scope_kind = 'workspace') = (workspace_id IS NOT NULL))
);
```

**Column rationale for the non-obvious fields.** `revoke_epoch` exists so an in-flight mutation can detect revocation before commit without a second round trip. `generation` on both state tables fences the reconciler: an effect completing against a stale generation is discarded rather than applied. `observed_state` is separate from `desired_state` because the durable record must distinguish "operator asked for this" from "the network actually reflects it", which is what makes crash recovery decidable. `endpoint_generation` binds a confirmation to the exact public address it was given, so a provider address change invalidates confirmations instead of silently redirecting a sender.

**Cleanup and ownership.** Ingress bindings are removed transactionally when their subject is deleted (a webhook trigger delete or bridge instance removal deletes the binding in the same transaction); a binding whose subject vanished is treated as absent by reads and swept on reconcile.

**Secret material (N-002).** Provider credentials reuse the **existing extension secret-binding contract** — the connectivity extension declares its env keys and the operator binds them like any extension secret. The gateway creates **no second secret owner** and stores no provider credential ref of its own. The only gateway-owned secret material is device credential hashes (in-table) and, on client machines, the local credential file. The vault `gateway` namespace is added solely for future gateway-owned secrets and is **not required by the MVP** — it is dropped from this revision to avoid an unused namespace.

**Ephemeral state.** Pairing artifacts and stream tickets remain in-memory, single-use, bounded, and TTL-expired: they must not survive a restart (the operator re-mints), and persisting them would create rows requiring a sweeper.

#### Side-table-vs-JSON decision

- Device sessions, provider identity, per-tier activation, surface exposure, and ingress bindings are **typed side tables with typed columns**, never JSON bags: every one is matchable, transitioned, or reconciled state (authenticate by `token_hash`, fence by `generation`, restore by `desired_state`, authorize by `workspace_id`). JSON metadata is forbidden for this state for the same reason `task_runs` uses columns.
- **JSON is used nowhere in this feature's schema.** No opaque metadata blob is introduced.

### API Endpoints and Tier Route Matrices

Handlers are shared `BaseHandlers` methods. The route matrix is selected by `httpapi.WithSurfaceSet(tier)` so a route physically cannot exist on the wrong listener (B-001, B-002).

**Private tier listener** — device-auth gated, plus these pre-auth routes and nothing else:
- `POST /api/gateway/pairings/redeem` — **pre-auth, private tier only.** 200 `IssuedCredential` (+ sets the browser cookie); 401 `gateway_pairing_invalid`; 410 `gateway_pairing_expired`; 409 `gateway_pairing_spent`.
- The SPA pairing gate assets (static, no data).

Authenticated private-tier routes:
- `GET /api/gateway/status` (also folded into `GET /api/status`) — 200 `GatewayStatus`.
- `POST /api/gateway/surfaces` — enable/disable a surface on a tier; `(operator_ui, public)` requires a consent token. 200; 409 `gateway_exposure_refused`; 428 `gateway_consent_required`.
- `POST /api/gateway/providers/:name/enable` / `POST /api/gateway/providers/:name/disable` — 200; 428 `gateway_digest_confirmation_required`; 409 `gateway_provider_trust_stale`; 403 on untrusted or workspace-scoped source.
- `POST /api/gateway/pairings` — mint. **Trusted surfaces only** (UDS, or an authenticated private-tier device). 201; 429 `gateway_pairing_limit`.
- `GET /api/gateway/devices`, `PATCH /api/gateway/devices/:id`, `DELETE /api/gateway/devices/:id` — 200 with `changed` in the body; 404 `gateway_device_not_found`.
- `POST /api/gateway/stream-tickets` — 201 `StreamTicket`.
- `POST /api/gateway/ingress-bindings`, `DELETE /api/gateway/ingress-bindings/:subject_kind/:subject_id` — 200/201; 409 when public ingress is off; 403 `gateway_ingress_forbidden` on cross-workspace attempts; 404 `gateway_ingress_subject_not_found`.
- `GET /api/gateway/audit` — 200 `AuditReport`.
- The operator/management API per the Remote Operation Matrix.

**Public tier listener** — the complete route inventory, nothing else:
- `POST /api/webhooks/global/:endpoint` and `POST /api/webhooks/workspaces/:workspace_id/:endpoint` — unchanged handlers, registered ahead of device auth, carrying their own HMAC/freshness/replay verification.
- Bridge inbound callback paths for bound instances.
- The endpoint-verification challenge path (nonce only).
- **When and only when `(operator_ui, public)` is consented:** the SPA assets and the authenticated operator/management API. **No pairing mint route and no pairing redeem route ever exist on this listener** (B-001) — a new device is admitted only through the private tier or local access.
- Everything else returns the uniform not-found response; the SPA fallback is disabled on this listener unless the operator UI surface is on.

**UDS** registers the gateway management subset (status, surfaces, providers, pairings, devices, audit, ingress bindings) for local operator and agent use, and keeps its existing superset unchanged and unexposed.

## Safety Invariants

Exposure, authentication, and provider paths are concurrency- and ownership-sensitive. The following invariants hold:

1. The daemon **never binds a non-loopback address**; only connectivity providers terminate external transport. Tier listeners bind loopback.
2. **Tier listeners never grant loopback trust.** Every request either authenticates as a device session or self-verifies (webhook HMAC). Being local on a tier listener grants nothing.
3. **Tier is a property of the listener, never a header.** Route matrices are selected at listener construction; the public listener contains no operator route unless `(operator_ui, public)` is consented, and contains **no pairing mint or redeem route under any configuration**.
4. **Exposure intent has exactly one authority: the database.** Configuration contributes only static tunables and an immutable ceiling (`gateway.enabled`); no config key duplicates runtime intent.
5. **Every transition is fenced by `generation`.** Desired state is written in one SQLite transaction that bumps `generation`; any effect completing against a stale generation is discarded, never applied.
6. **The reconciler applies effects in a fixed order with compensation** — persist desired → bind listener → establish provider → verify endpoint → advertise — and unwinds in reverse on failure, so no intermediate state is externally reachable.
7. **Boot reconciles before advertising.** `Reconcile` runs before any endpoint is advertised or any tier listener accepts external traffic; a crash therefore restores the operator's last desired state and never re-exposes a disabled surface.
8. **An endpoint is advertised only after challenge verification** proves it reaches this daemon's assigned tier listener; verification failure marks the provider degraded and advertises nothing while a generation-fenced recovery retains the staged listener for another proof.
9. **Provider trust is re-derived from the live extension registry** on every enable and boot; a digest mismatch fails closed and requires re-consent.
10. **Pairing artifacts are single-use, bounded, and TTL-expired**; redeem is atomic — exactly one redeemer wins a race and the rest receive `ErrPairingSpent`.
11. **Credential verification is constant-time**; the raw credential exists only in transit and in client-local storage, never in a daemon table or log.
12. **Revocation terminates live connections before returning**: the revoke transaction bumps `revoke_epoch`, then `ConnectionRegistry.CancelDevice` cancels every registered stream for that device, and only then does the call return.
13. **In-flight mutations revalidate before commit** via `RevalidateForCommit`; a device revoked mid-call fails the transaction rather than committing a partial write.
14. **Stream tickets are single-use and short-TTL**; a credential never appears in a stream URL, log, or browser history, and every reconnect acquires a fresh ticket.
15. **Enabling the public operator UI requires fresh consent each time**; disabling any surface is immediate, unconditional, and survives restart.
16. **Ingress bindings authorize through the canonical workspace boundary**: the subject's owner is resolved server-side before mutation, client-supplied scope is never trusted, and a confirmation is bound to an `endpoint_generation`.
17. **The gateway owns no queue, scheduler, or claim primitive.** Task claim/heartbeat/complete/fail/release stay `task.Service` primitives reached over UDS; `ClaimNextRun` remains the sole claim authority and claim-token fencing is untouched.
18. **Secrets never cross surfaces unredacted** — device credentials, pairing artifacts, stream tickets, provider credentials, and raw `compozy_claim_*` values appear only as hashes, refs, or truncated forms in logs, status, events, audit output, and every tier listener response.

## Integration Points

- **Connectivity provider (bundled Tailscale):** authenticates to the operator's tailnet via `tsnet`; establishes a tailnet listener and a Funnel listener; forwards each to its assigned loopback tier listener. Credentials ride the existing extension secret-binding contract. Failure surfaces as `degraded` with backoff via the extension supervisor.
- **Automation/webhooks:** verification unchanged; the gateway supplies reachability, per-trigger public delivery URLs, and binding lifecycle.
- **Bridges:** inbound callbacks bind to the public tier via `gateway_ingress_bindings`; bridge health gains ingress reachability.
- **Extension registry:** the source of truth for provider install source and control digest on every enable and boot.

## Impact Analysis

| Component | Impact Type | Description and Risk | Required Action |
| --- | --- | --- | --- |
| `internal/gateway` (new) | new | Policy, reconciler, devices, connection registry, provider supervision, verification, audit, status. High risk (security surface). | Build with the decided file split; full failure-path coverage. |
| `internal/daemon` | modified | `bootGateway`, tier-listener factory + port publishing, reconcile-before-advertise, shutdown ordering. High. | Mirror existing server-factory wiring. |
| `internal/api/httpapi` | modified | `WithDeviceAuth`, `WithSurfaceSet`, tier route matrices, cookie handling. High (auth-critical). | Mirror `WithResourceOperatorAuth`; per-tier route tests. |
| `internal/api/udsapi` | modified | Gateway management subset; superset stays unexposed. Low. | Register subset; parity test. |
| `internal/api/core` | modified | `GatewayService` interface, `BaseHandlers` methods, gateway→contract mapping, `StatusForGatewayError`, status extension. Medium. | Implement + map. |
| `internal/api/contract` | modified | Gateway wire DTOs. Medium (co-ship). | `make codegen`. |
| `internal/api/spec` | modified | Operation registry entries + auth metadata for every gateway route. Medium. | Register; parity suite. |
| `internal/extensionprotocol` / `internal/extension` | modified | `connectivity.provider` surface + service methods + consumer; workspace-source rejection. Medium. | One-file surface add + consumer. |
| `extensions/connectivity/tailscale/` (new) | new | Bundled `tsnet` provider; binary weight. Medium. | Embed + re-exec; enroll in `bootExtensions`. |
| `internal/store/globaldb` | modified | Schema fragment + appended Goose migration + `queries/gateway.sql` + generated sqlc + `GatewayRepo`. Medium. | `eng-schema-migration`; `make codegen`. |
| `internal/events` | modified | Gateway event names + component + registry entries. Low. | Register in `components.go`, `gateway.go`, `registry_base.go`. |
| `internal/tools/builtin` | modified | `compozy__gateway` descriptor, binding, availability, generated catalog. Medium. | Descriptor + catalog regen. |
| `internal/config` | modified | `[gateway]` section, overlay, validation, home paths. Low. | Copy `daemon.go` pattern. |
| `internal/settings` | modified | `SectionGateway` live-apply classification. Low. | Classify Live vs RestartRequired. |
| `internal/cli` | modified | Target-aware transport, client credential store, profiles, command groups, operation matrix enforcement, scheme-aware `open`. Medium. | Single chokepoint + new groups. |
| `internal/automation` / `internal/bridges` | modified | Delivery URL projection, transactional binding cleanup on subject delete. Low. | Add cleanup + URL surface. |
| `web/` | modified | Gateway settings, device list, pairing, consent, audit; stream-ticket acquisition for remote sessions. Medium. | See Web/Docs Impact. |
| `packages/site` | modified | Remote-gateway runbook + threat model; config + CLI reference. Low. | New docs pages. |
| `magefiles/boundaries.go` | modified | `internal/gateway` rules. Low. | Mirror marketplace block. |
| `skills/compozy/` | modified | Agent-facing gateway paths + operation matrix. Low. | Update built-in skill. |

## Testing Approach

- **Frameworks/harness:** Go `testing`, `t.Run("Should …")` subtests, `t.Parallel` by default (never on env-mutating cases), `-race`/`CGO_ENABLED=1`, table-driven. Fakes sit only at I/O boundaries: a fake clock for TTL/epoch, and a **real subprocess** conformance fixture for the connectivity provider (an in-memory fake is used only for supervisor-logic unit cases, never as the sole coverage of the protocol — L-007).
- **Test placement:** every case names its invariant, owning layer, and the canonical suite file it extends; the same invariant is not re-asserted at a second layer unless the layers fail differently, and that difference is stated (per the repo's test-placement rule and L-011).
- **Unit** covers `internal/gateway` services and every error path, including fence/epoch races, verification rejections, and redaction.
- **Integration** covers boundaries: per-tier route matrices (a route absent on the wrong listener), device auth over a tier listener, webhook survival ahead of the auth gate, reconcile-after-crash, revocation canceling live streams, provider enable→verify→advertise, ingress cross-workspace denial, CLI remote transport, HTTP/UDS parity, and the migration fresh/reopen/ahead/integrity/equivalence suites.
- **Extension conformance** exercises the real JSON-RPC/stdio provider protocol: method negotiation, malformed payloads, permission/digest enforcement, crash and teardown, and output redaction.
- **Security coverage** asserts byte-level that no raw `compozy_claim_*`, device credential, pairing artifact, or stream ticket crosses HTTP, UDS, CLI, SSE, channel, event, or memory surfaces — only hash forms.
- **E2E** follows the user journeys (pair-and-use, webhook→Loop, revoke-terminates-stream, SSH connect, audit remediation).
- **Environment:** integration/E2E use a temp `COMPOZY_HOME`, tier listeners on `:0`, and the fake/real fixture provider; the bundled `tsnet` provider runs only in a tagged manual lane, never in CI.

Concrete cases live in `_tests.md`.

## Development Sequencing

### Build Order

1. **Config + home paths + boundaries** — `[gateway]` section, gateway state paths, `internal/gateway` boundary rules. No dependencies.
2. **Policy + desired/observed schema + reconciler** — Global DB fragment, appended migration, `queries/gateway.sql`, generated sqlc, `GatewayRepo`, state machine, `Reconcile`. Depends on 1.
3. **Tier listeners owned by the daemon** — `gatewayServerFactory`, `WithSurfaceSet` route matrices, port publishing, boot order (reconcile before advertise), shutdown ordering. Depends on 2.
4. **Device sessions + pairing + connection registry** — credential format, hashing, epoch, in-memory artifacts/tickets. Depends on 2.
5. **Device-auth middleware + stream tickets + browser cookie flow** — `WithDeviceAuth`, redeem route (private only), ticket acquisition and reconnect, web consumer migration. Depends on 3, 4.
6. **`connectivity.provider` surface + consumer + supervisor + verification protocol** — protocol const, adapter, challenge verification, trust re-derivation, digest consent. Depends on 2.
7. **Bundled `tsnet` provider** — `extensions/connectivity/tailscale/`, enrollment in `bootExtensions`. Depends on 6.
8. **Webhook/bridge public reachability** — delivery URL projection, ingress bindings with workspace authorization, transactional cleanup, bridge health. Depends on 5, 7.
9. **Remote CLI profiles + operation matrix** — target-aware transport, client credential storage, `[[gateway.connections]]`, command groups, `ErrLocalOnlyOperation` enforcement, scheme-aware `open`. Depends on 4, 5.
10. **SSH connection method** — `compozy connect ssh`, launch/reuse/teardown. Depends on 9.
11. **Generated, native-tool, and event surfaces** — `internal/api/spec` operations + auth metadata + parity suites; OpenAPI + generated TS + site reference via `make codegen`; `compozy__gateway` descriptor → binding → availability → generated native catalog; `internal/events` component + names + `registry_base.go` entries with payload/redaction mapping and durable-append-before-publish. Depends on 3–9.
12. **Self-audit + web surface + docs** — `Auditor`, gateway settings/device/pairing/consent/audit UI, runbook + threat-model docs. Depends on 2–11.
13. **QA planning + QA execution** — the trailing pair.

### Technical Dependencies

- `tailscale.com/tsnet` added via `go get` (never hand-edited into `go.mod`).
- `make codegen` regenerates OpenAPI, generated TypeScript, sqlc output, and the native tool catalog; `make codegen-check` gates drift.
- The Global DB migration is append-only with refreshed `atlas.sum` (`eng-schema-migration`).

## Monitoring and Observability

- **Event registration (B-011):** add `ComponentGateway = "gateway"` to `internal/events/components.go`, event name constants to a new `internal/events/gateway.go`, and registry entries in `internal/events/registry_base.go` using the existing `global(...)`/`success`/`warning`/`failure`/`notify` wrappers, extending `internal/events/registry_test.go`. Events: `gateway.exposure.changed`, `gateway.consent.recorded`, `gateway.device.paired`, `gateway.device.revoked`, `gateway.provider.state_changed`, `gateway.endpoint.verified`, `gateway.endpoint.rejected`, `gateway.ingress.bound`, `gateway.ingress.unbound`, `gateway.audit.run`.
- **Durable append before publish** — events persist before any live broadcast, per the existing observability contract.
- **Attribution (N-003):** remote actions reuse the **canonical `actor_kind` / `actor_id`** correlation fields (`actor_kind='operator_device'`, `actor_id=<device id>`); no new event column and no migration. Local actions keep their existing actor values, so remote versus local remains distinguishable and filterable by the existing query paths.
- **Redaction mapping:** each gateway event declares which payload fields are redacted; the byte-level security suite covers them.
- **Status fields:** active tiers/surfaces, advertised addresses, per-surface reachability, device count, per-tier provider health — in `GET /api/status` and `compozy daemon status`.

## Extensibility Integration Plan

- **New provide surface `connectivity.provider`** in `internal/extensionprotocol/capabilities.go` with daemon→provider service methods (`connectivity/establish|status|teardown`); public, so third parties may implement it, gated by install-source tiers plus digest confirmation re-derived on every enable and boot (ADR-009). Workspace-scoped sources are rejected for this surface.
- **Host API:** no extension→daemon Host API method is added — providers report through the daemon-initiated `connectivity/status` call, so the closed Host API method set (currently **95** methods; the glossary's "87" is stale and is corrected in the same change) is unchanged.
- **Consent:** a dedicated consent area names reachability control; digest confirmation reuses the existing Network-Requirement-Confirmation mechanism.
- **Provider secrets** reuse the existing extension secret-binding contract — no second secret owner (N-002).
- **Bundled extension** `extensions/connectivity/tailscale/` (embed + re-exec), enrolled via `EnsureManagedInstall` in `bootExtensions`; scaffold template `scaffold_templates/connectivity-provider-{go,ts}/` plus SDK authoring helpers.
- **Hooks (N-004):** **no gateway hooks are added, and no hook tails an event table.** Gateway transitions are ledger-only. If hooks are wanted later, they dispatch at the owning transition call site, never from the event store.
- **Unchanged consumers:** bridges gain a public callback binding path; webhook triggers gain reachable delivery URLs; skills/tools/automation/MCP/registries/bridge SDKs are otherwise unaffected.

## Agent Manageability Plan

- **CLI verbs** (structured `-o json`): `compozy gateway status|surface enable|surface disable|provider enable|provider disable|audit`, `compozy pair mint|redeem`, `compozy device list|rename|revoke`, `compozy connect add|list|use|remove`, `compozy connect ssh`.
- **HTTP/UDS parity:** every management verb maps to one `BaseHandlers` method served on UDS and the private tier; parity suites assert identical behavior.
- **Operation matrix** is published in CLI help, site docs, and the built-in skill, and enforced by `ErrLocalOnlyOperation`.
- **Deterministic errors:** `gateway_exposure_refused`, `gateway_consent_required`, `gateway_pairing_invalid|expired|spent|limit`, `gateway_device_not_found`, `gateway_stream_ticket_invalid`, `gateway_digest_confirmation_required`, `gateway_provider_trust_stale`, `gateway_endpoint_unverified`, `gateway_provider_degraded`, `gateway_ingress_forbidden`, `gateway_ingress_subject_not_found`, `gateway_local_only_operation`.
- **Native tool (B-011):** `compozy__gateway` with a full descriptor → binding → dependency → availability path and regenerated native catalog (`internal/tools/builtin/testdata/native-tool-catalog.json`); read operations (status, audit) always available, mutations gated by permission mode.
- **Self-audit** returns machine-readable findings with stable ids, severity, and remediation.

## Config Lifecycle

New `[gateway]` section (`internal/config/gateway.go`, patterned on `daemon.go`), overlay in `internal/config/merge_gateway.go`, validated in `validateCore`, defaults in `DefaultWithHome`:

```
gateway.enabled                   bool     false  -- immutable ceiling: may the gateway operate at all; false = local-only. Never a runtime intent duplicate.
gateway.private_port              int      0      -- loopback port for the private tier listener; 0 = OS-assigned
gateway.public_port               int      0      -- loopback port for the public tier listener; 0 = OS-assigned
gateway.pairing.ttl               duration 5m     -- pairing artifact lifetime
gateway.pairing.max_pending       int      8      -- bound on concurrent pending artifacts
gateway.stream_ticket.ttl         duration 30s    -- single-use stream ticket lifetime
gateway.auth.rate_limit.window    duration 60s    -- failed-auth counting window per source
gateway.auth.rate_limit.max_fails int      10     -- failures before lockout; local access exempt
gateway.verify.timeout            duration 10s    -- endpoint challenge deadline
gateway.verify.public_dns_resolver string  1.1.1.1:853 -- public DNS-over-TLS resolver used instead of host MagicDNS
```

- **`gateway.public_ui.enabled` is deliberately absent (B-005).** Runtime exposure intent lives only in `gateway_surface_exposure`; config carries no key that could contradict it. `gateway.enabled` is a ceiling: with it false, every transition refuses, and turning it false disables all tiers on reload.
- **Overlay/merge:** pointer-nil "unset" semantics; strict unknown-key rejection means every key is modeled or config load fails.
- **CLI `config set`:** `gatewayConfigSetPathKinds()` registered in `mergeConfigSetValueKinds`; live-apply via `SectionGateway` in `internal/settings` (ceiling and TTL/rate-limit changes are `Live`; port changes are `RestartRequired`).
- **Docs:** `packages/site/content/docs/configuration/config-toml.mdx`, a new `operations/remote-gateway.mdx` runbook + threat model, and generated CLI reference.
- **Tests:** defaults, validation (bad ports/durations/negative bounds), overlay unset semantics, `config set` classification, live-apply classification, and a case asserting no config key can enable a surface.

## Compozy Impact Audit

- **Native tools:** new `compozy__gateway` tool ID with descriptor, binding, dependencies, availability diagnostics, I/O schemas, risk flags, permission-mode gating, and a regenerated `native-tool-catalog.json`. No existing tool ID, toolset, or schema digest changes.
- **Extensibility and hooks:** new `connectivity.provider` provide surface + service methods; no Host API additions (set stays 95); new consent area + digest confirmation reusing the existing mechanism; provider secrets via the existing extension secret-binding contract; bundled `tailscale` extension + scaffold + SDK helpers; **no hooks added and no event-table tailing**; MCP sidecars, registries, and bridge SDKs unchanged.
- **Workspace data isolation:** device sessions, provider identity/activation, and surface exposure are **operator-global with no `workspace_id`** and stay out of workspace-scoped read paths. **`gateway_ingress_bindings` is the exception and is workspace-aware by design** (B-006): it stores the subject's resolved `scope_kind`/`workspace_id`, the owner is resolved server-side (never from client input), agent callers are authorized through the canonical workspace boundary before mutation, and cross-workspace bind/unbind is denied with `gateway_ingress_forbidden`. List/read/cache/SSE/event paths for bindings filter by the resolved scope; global-scope webhooks carry `scope_kind='global'` and no workspace id. Attribution added to events uses existing global actor fields, not workspace-scoped data.
- **Official Compozy skill:** `skills/compozy/` updated with gateway management verbs, the remote operation matrix, native tool usage, and the deterministic error codes.

## Technical Considerations

### Key Decisions

- **Loopback-only daemon, provider-forwarded tier listeners, tier-by-listener** (ADR-007), with listener construction owned by the composition root. Rejected: daemon binds the managed address; a single listener with a trusted tier header.
- **Opaque revocable credential + stream tickets + connection registry + revoke epoch** (ADR-008). Rejected: proof-of-possession now; OAuth for public now.
- **Public `connectivity.provider` surface with challenge-based endpoint verification and registry-derived trust; bundled `tsnet` provider** (ADR-009). Rejected: bundled-only; sidecar Tailscale; probe-only verification.
- **Durable desired/observed state machine with generation fencing, database as the single intent authority** (ADR-007/010). Rejected: in-memory transition lock; config-plus-DB dual authority.
- **Global DB + existing extension secret bindings + client-local CLI credentials** (ADR-010) and **one `internal/gateway` domain package** (ADR-011).
- **Bounded remote operation matrix** (ADR-005). Rejected: "every verb remote", which would have exposed the UDS-only superset or made claim paths remotely reachable.

### Known Risks

- **Auth and tier-routing correctness is load-bearing.** Mitigation: invariants are individually tested; integration asserts route absence on the wrong listener, revocation canceling live streams, and reconcile-after-crash.
- **Reconciler complexity.** A desired/observed machine with compensation is more code than a naive toggle. Mitigation: fixed effect order, generation fencing, and fault-injection tests at every effect boundary.
- **`tsnet` weight and maintenance.** Mitigation: pinned dependency; the provider seam allows alternatives without core changes.
- **Webhook durability bounded by daemon uptime** (PRD non-goal). Mitigation: documented at every delivery-URL surface with the sender-side redelivery path.
- **Browser cookie flow interacts with the existing origin protection.** Mitigation: SameSite plus the existing cross-origin protection, with integration coverage of the remote origin case.

## Assumptions and Defaults

- The operator holds their own Tailscale account; the bundled provider never uses a Compozy-owned account.
- Remote UI is the daemon serving its own SPA at the reachable address; no hosted multi-backend SPA and no backend picker.
- Default posture is fully local-only (`gateway.enabled = false`); nothing is reachable until a tier is enabled and a device is paired.
- Tier listeners default to OS-assigned ports resolved back from the listener.
- Pairing artifacts and stream tickets do not survive restart by design.
- The SSH method requires Compozy installed on the remote host, uses the operator's existing SSH credentials/config, and fails closed on a changed host key.
- TTL and rate-limit defaults (5m pairing, 30s ticket, 10 fails/60s, 10s verify) are tunable via `[gateway]`.
- The Host API method count is 95; the stale glossary figure is corrected in this change.

## QA Lifecycle (N-006)

The trailing QA pair operates on the committed `docs/qa/` tree and must: bootstrap a fresh lab via `eng-qa-bootstrap` with a unique `COMPOZY_HOME`, unique daemon/tier ports, and unique tmux socket; derive `COMPOZY_WEB_API_PROXY_TARGET` from the bootstrap manifest (never hardcoded); run config writes sequentially per home; register long-lived lab processes at `<QA_OUTPUT_PATH>/qa/pids/<name>.pid`; add or reset the affected `docs/qa/scenarios/` entries for every user-visible change in this feature and walk each to a recorded verdict; and end every terminal path with the manifest teardown, citing `teardown.json` with `"clean": true`.

## Architecture Decision Records

- [ADR-001: First-party managed connectivity without hosted infrastructure](adrs/adr-001.md)
- [ADR-002: Single-operator identity with per-device pairing](adrs/adr-002.md)
- [ADR-003: Exposure policy in core; connectivity mechanisms as provider extensions](adrs/adr-003.md)
- [ADR-004: Public tier surface set — ingress by default, consented operator UI, never pairing](adrs/adr-004.md)
- [ADR-005: Remote client methods and the bounded remote operation matrix](adrs/adr-005.md)
- [ADR-006: Fail-closed exposure with agent-operable self-audit](adrs/adr-006.md)
- [ADR-007: Daemon-owned tier listeners with a durable desired/observed state machine](adrs/adr-007.md)
- [ADR-008: Opaque revocable credentials, stream tickets, and connection-registry revocation](adrs/adr-008.md)
- [ADR-009: `connectivity.provider` surface, challenge-based verification, registry-derived trust](adrs/adr-009.md)
- [ADR-010: Gateway state ownership — Global DB schema, secret ownership, client-local CLI credentials](adrs/adr-010.md)
- [ADR-011: `internal/gateway` domain package, interfaces in `internal/api/core`](adrs/adr-011.md)
