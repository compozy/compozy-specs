# PRD: Remote Gateway

## Overview

Compozy today is locked to the machine it runs on. The operator can only use the product while sitting at that machine: there is no way to authenticate from another device, external services cannot deliver events to a locally running daemon, and every messaging bridge forces the operator to build and maintain their own proxy infrastructure with a hand-managed public address. The result is that an "agent operating system" — a long-running system whose whole point is autonomous work continuing while the operator is away — is unusable precisely when the operator is away.

The Remote Gateway makes a Compozy daemon safely reachable from outside its machine, under the operator's control, in layered tiers:

- **Private overlay**: the operator's own devices (phone, laptop, work machine) reach the full product from anywhere over a personal overlay network, with nothing to install on the daemon machine beyond Compozy itself.
- **Public ingress**: the daemon gets a stable public web address so external services — starting with repository webhooks — can deliver signed events that trigger automation and Loops. By default the public address serves delivery ingress only.
- **Remote clients**: the CLI gains connection profiles to operate remote daemons with full parity, and an SSH connection method brings up and reaches a daemon on any machine the operator can already SSH into, with zero network exposure.

It is for the single operator who self-hosts Compozy and for the agents that manage it on their behalf. Its value: the two scenarios that motivated it — "full access to my agent fleet from anywhere" and "a repository push starts a Loop on my machine" — become first-class product journeys instead of networking projects, and they are safe by construction: exposure without authentication is impossible.

## Goals

What ships when this feature exists, stated as observable behavior:

- The operator reaches the complete product (sessions, live streams, approvals, settings) from any paired personal device, anywhere, over the private overlay — with no networking software installed or configured on the daemon machine beyond Compozy.
- External services deliver signed events to a stable public address and those events trigger automation, including starting Loops; configuring a sender is a copy-paste of a displayed delivery URL.
- Messaging bridges bind their inbound callbacks to the same public address, eliminating operator-built proxy infrastructure per bridge.
- Exposing any surface without the authentication model active is impossible — every such attempt is refused with a diagnostic naming the exact cause and fix, and no override exists.
- Every remote access point (device, CLI profile) is individually visible, named, and revocable with immediate effect.
- The CLI operates remote daemons through named connection profiles with the same commands, output, and streaming as local use.
- Any machine the operator can SSH into becomes reachable through a single connect command that launches or reuses the daemon there, exposing nothing publicly on either end.
- Agents inspect, configure, pair, revoke, and audit all of the above through structured command surfaces without the web UI.
- Third parties extend connectivity (new ways to make a daemon reachable) by shipping provider extensions, without changes to the runtime.

## Success Criteria

Verifiable against the shipped product, from the outside:

- Fresh install to remote access: enable the overlay, pair a phone, and open the full UI from an off-network device using only settings/CLI flows — no configuration file editing.
- Webhook-to-Loop round trip: create a webhook trigger bound to a Loop, paste its displayed delivery URL into a repository, push, and observe a Loop run whose origin attributes the trigger and delivery; the sender's own delivery log records success.
- Revocation: revoking a device from the device list terminates that device's access immediately, including live streams; the device lands on an explicit "access ended" state.
- Teardown: one action returns the daemon to local-only; the self-audit confirms the posture; a restart re-exposes nothing that was turned off.
- Fail-closed: every attempt to reach non-local exposure without the authentication model active is refused with an actionable message; following the named fix makes the same action succeed.
- Audit loop: the self-audit reports a finding, the operator or an agent applies its remediation, and a re-run reports the finding cleared.

## User Stories

Full catalog with acceptance criteria and edge cases: [Full user stories](_user_stories.md).

- US-001–US-004 — Gateway status and exposure modes: posture at a glance, overlay enablement, fail-closed refusals, instant teardown.
- US-005–US-008 — Device pairing and management: one-time pairing, device list operations, revoked-device experience, agent-facing device management.
- US-009–US-010 — Private overlay: full product from anywhere; overlay reachability is not authentication.
- US-011–US-013 — Public address: stable ingress address, opt-in operator surface with consent, hostile-probing posture.
- US-014–US-018 — Webhook ingress: per-trigger delivery URLs, signed-delivery verification and dispatch, push-to-Loop journey, bridge callback binding, ingress/operator auth separation.
- US-019–US-021 — Remote CLI: profile pairing, full command parity, headless agent operation.
- US-022–US-023 — SSH connection: launch/reuse and local access, zero-exposure guarantee.
- US-024–US-025 — Provider extensions: provider contract for developers, trust-gated installation.
- US-026–US-028 — Security and audit: self-audit, secret redaction, device attribution.

## Core Features

1. **Gateway status and named exposure posture** — One surface (settings panel and status command) that always answers "how is my daemon reachable right now": active modes, every advertised address, which surfaces are reachable at each, device count, provider health. Interacts with every other feature: all of them report into it.

2. **Device pairing and device management** — One operator identity across many devices. A trusted surface (daemon machine's local UI or CLI) mints a one-time, short-lived pairing artifact (scannable code and copyable link); the new device becomes a named, individually revocable device session. The device list shows origin and last activity; revocation is immediate. The daemon machine's local access is the root of trust and the recovery path.

3. **Private overlay access (bundled provider)** — The operator enables overlay reachability from gateway settings by authorizing with their own overlay-network provider account (first supported provider: Tailscale). The daemon becomes reachable at a stable private address for paired devices, carrying the full product surface, with nothing else to install on the daemon machine. Ships as a first-party bundled provider extension (see feature 8).

4. **Stable public ingress address with per-surface opt-in** — The operator enables a stable public web address through the provider. By default it serves delivery ingress only (webhook triggers, bridge callbacks). The operator may additionally opt the operator UI/API into public reach behind pairing, through an explicit consent step restated on every enable. Disabling any surface is immediate.

5. **Webhook delivery reachability** — Every webhook trigger displays its exact public delivery URL, copy-paste ready. Signed deliveries are verified (signature, freshness, replay dedupe) before any side effect, then dispatched to automation — including starting Loops — with origin attribution visible on the resulting run. Bridges bind their inbound callback URLs to the public address per instance, with confirmation, and ingress reachability becomes part of bridge health. This feature makes the existing automation webhook and bridge systems reachable; it does not replace their verification or dispatch semantics.

6. **Remote CLI connection profiles** — The CLI pairs with a remote daemon as a named connection profile (its own entry in the device list). Every command then operates the remote daemon with identical behavior, structured output, and streaming; the remote target is always explicit; auth failures and reachability failures are deterministically distinct.

7. **SSH connection method** — `connect to a machine I can already SSH into`: using the operator's existing SSH credentials and configuration, Compozy launches the daemon on the remote machine (or verifiably reuses a running one), makes it available locally for the duration of the connection, and tears down only what it started. Requires Compozy installed on the remote machine. Exposes nothing publicly on either end.

8. **Connectivity provider extension seam** — Exposure policy (modes, auth, fail-closed, per-surface decisions, audit, teardown) lives in the runtime and is never delegated. The mechanisms that produce reachability — join an overlay, establish a public address — are provider extensions with a declared provide surface, supervised lifecycle, and health reporting. Endpoints a provider reports are advertised only after the runtime verifies them. The first provider ships bundled and first-party; third-party providers install under trust gating with explicit operator consent.

9. **Fail-closed posture and security self-audit** — The unsafe state is unrepresentable: no path exposes a surface without authentication active, and no override flag exists. An on-demand self-audit (UI and structured command) reports exposure posture, auth state, device inventory highlights, provider health, and ranked findings with concrete remediation — consumable by humans and agents.

## Business Rules

### Exposure

1. A fresh install has zero remote reachability; local-only is the permanent default.
2. Exposure is decided per surface: the private overlay carries the full product surface for paired devices; the public address serves delivery ingress only unless the operator opts the operator surface in.
3. Any transition that would make a surface reachable beyond the local machine while the authentication model is inactive is refused with a diagnostic naming cause and fix. No flag, environment value, or configuration combination bypasses this.
4. A direct network bind onto an operator-managed network is subject to the same rules: authentication active, pairing gate served, no exceptions.
5. Disabling any exposure takes effect immediately. A restart restores the operator's last configured intent and never re-exposes a surface that was turned off.
6. Network reachability is never authentication: a device that can reach any address still sees only the pairing gate until it holds a device session.
7. Enabling the operator surface on the public address requires an explicit consent step that states what becomes publicly reachable; consent is required again on every enable.

### Identity and devices

8. There is exactly one operator identity; no user accounts, roles, or guest access exist.
9. A new device is admitted only through a pairing artifact that is: minted from an already-trusted surface, single-use, and valid for at most 5 minutes (default).
10. At most 8 pairing artifacts (default) may be pending at once; minting beyond the bound refuses with a clear message.
11. Every paired device is a named device session listing origin, creation time, and last activity; sessions are individually revocable and revocation takes effect immediately, including on live connections and streams.
12. The daemon machine's local access is the root of trust: it can always mint pairings and revoke devices; losing all remote devices is recovered locally. No password or memorized-secret path exists.
13. The public pairing gate never mints pairing artifacts; minting is exclusive to trusted surfaces.
14. Failed authentication attempts are rate-limited per source; local access is exempt from that limiting.

### Public ingress and deliveries

15. Every delivery endpoint verifies signature, timestamp freshness, and delivery-identity uniqueness before any side effect; a failed verification produces no side effects, is observable to the operator, and reveals nothing to the sender beyond the failure.
16. Delivery verification secrets are distinct credentials from operator authentication: an operator session cannot substitute for delivery verification, and a valid delivery grants nothing beyond dispatching its own trigger.
17. Ingress is rate-limited per endpoint and per source; over-limit outcomes are retry-appropriate and never silent drops.
18. Delivery reachability is bounded by daemon uptime: there is no store-and-forward. The limitation and the sender-side redelivery path are documented wherever delivery URLs are displayed.
19. A bridge instance binds its inbound callback to the gateway public address only through per-instance confirmation; bound bridges include ingress reachability in health; a public-address change flags every dependent (trigger URL displays, bound bridges) for re-confirmation.

### Clients

20. Remote CLI profiles carry full command parity, including streaming and structured output; the remote target is always explicit in the interaction; authentication errors and reachability errors have distinct, stable identities.
21. Work started from a remote client never dies with the client's connection: execution is detached, and a reconnecting client observes progress.
22. The SSH method uses the operator's existing SSH trust (keys, configuration, host verification); a changed host identity refuses the connection; it stops only daemons it started, and it never widens network exposure on either machine.

### Providers and extensibility

23. Exposure policy is never delegated to extensions: providers contribute reachability mechanisms and health only.
24. An endpoint reported by a provider is advertised only after the runtime verifies it; provider failure degrades affected reachability to a safe state — never half-exposed — with supervised restart and visible status.
25. Installing or enabling a third-party connectivity provider requires explicit operator consent that names its control over daemon reachability; untrusted install sources are blocked by default; an update that expands a provider's declared control requires re-consent.
26. The bundled first-party provider operates exclusively against the operator's own provider account; Compozy operates no account, relay, or infrastructure on the operator's behalf.

### Security and observability

27. Pairing artifacts, provider credentials, and delivery verification secrets never appear unredacted in logs, status output, events, audit reports, error messages, or diagnostics; only redacted or fingerprint forms appear anywhere.
28. The self-audit is available on demand through the UI and a structured command; findings carry stable identities, severity, and concrete remediation; "no findings" is an explicit result.
29. Every state-changing action performed remotely is attributed to its device session identity in the operational event ledger, alongside existing correlation fields; attribution survives revocation.
30. Every exposure-affecting setting participates in the daemon configuration lifecycle: declared, validated, documented, and reflected in status — configuration, runtime state, and documentation never disagree silently.

### Lifecycle states

- Pairing artifact: `minted → spent | expired`. No other transitions; a spent or expired artifact is never revivable.
- Device session: `active → revoked | expired`. Terminal states are permanent; a returning device requires a new pairing (new session identity).
- Public operator surface: `off ↔ on(consented)`. Every `off → on` passes through consent; `on → off` is immediate and unconditional.
- Connectivity provider: `installed → enabled → degraded → disabled`. `degraded` is a supervised failure state visible in status; `enabled → degraded` and back never passes through a half-exposed intermediate.

## User Experience

### Key personas

- **Operator** — self-hosts Compozy; wants their agent fleet in their pocket and external events flowing in. Competent, but must never need to be a network engineer for the happy path.
- **Agent** — operates the gateway headlessly for inspection, configuration, and repair.
- **External service** — delivers signed events; sees only the public ingress.
- **Provider developer** — extends connectivity through the provider contract.

### Primary flows

1. **First contact**: the gateway settings surface shows the exposure ladder with everything off and plain-language explanations of each tier. Status always answers "how am I reachable" — the same truth in UI and CLI.
2. **Enable overlay + pair a phone**: enable the overlay → authorize with the provider account → status shows the private address → mint a pairing (scannable code, plus a copyable link/text alternative) → phone opens the gate, pairs, and lands in the full UI. Minutes, no file edits.
3. **Webhook to Loop**: create/choose a webhook trigger → the trigger view shows its delivery URL and verification-secret setup together → paste both into the sender → push → watch the Loop run appear with its delivery attribution. The trigger view is honest when public ingress is off: it says so and links the enable path instead of showing a dead URL.
4. **Work-laptop CLI**: mint a pairing → complete it from the CLI on the other machine → a named profile exists; every command now carries an explicit remote-target indication.
5. **SSH into a server**: one connect command with a host the operator can already SSH into → daemon launched or reused → local UI/CLI work against it → disconnect tears down only what was started.
6. **Routine safety**: the device list is a living inventory (rename, revoke, last activity); the self-audit is one action away and its findings say exactly what to do; "Test connection" affordances validate the real access path a client would use — not merely a status ping — so a green check means the connection actually works.

### UI/UX considerations and accessibility

- The gateway surface presents named modes with plain-language risk framing, not raw toggles; the public-operator-surface consent step states concretely what becomes reachable.
- Every scannable pairing code has a copyable text equivalent (screen readers, remote terminals, machines without cameras).
- Status, device list, audit results, and consent flows meet the product's accessibility floor: keyboard operable, screen-reader legible, no color-only signaling.
- Truthful UI: reachability shown in the UI reflects verified runtime state, never provider claims alone; degraded states are visible, not masked.
- Failure UX is a designed surface: refusals name the fix; reachability, authentication, and verification failures are always distinguishable from each other everywhere they can occur.

### Onboarding and discoverability

- The feature is discoverable from the settings surface and from status output on a fresh install, both stating that everything is off by default.
- Each tier's enable flow teaches its trade-off inline (private overlay vs public address vs SSH), replacing the current documentation vacuum around exposure.
- Documentation ships with the feature: an operator runbook for each flow and a threat-model page stating what each tier exposes and what the fail-closed guarantees are.

## Agent and Operator Manageability Outcome

Everything a human can do in the gateway UI is available to agents and operators through structured, non-UI surfaces with deterministic behavior: inspect posture and addresses, enable/disable exposure surfaces, mint and revoke pairings, list and manage devices and profiles, read provider health, run the self-audit, and consume machine-readable findings. Errors carry stable identities so agents can branch on them ("re-pair needed" vs "unreachable" vs "refused by policy"). Remote manageability is itself part of the feature: an agent with a connection profile manages a remote daemon the same way a local one does. No capability of this feature is UI-only.

## Extension Ecosystem Expectation

Connectivity mechanisms are an extension point by design: the runtime owns exposure policy and publishes a provider contract; reachability mechanisms — including the first-party bundled one — are provider extensions with supervised lifecycles. Third parties are expected to add providers without runtime changes, subject to trust gating and explicit operator consent, because a connectivity provider influences how the daemon is reached. Existing extension surfaces (skills, hooks, tools, bridges, automation) are consumers of the new reachability, not modified contracts: bridges gain a public callback path, webhook triggers gain reachable URLs, and hook/automation semantics are unchanged.

## High-Level Technical Constraints

- Builds on the existing automation trigger system, bridge system, extension system, secret storage, operational event ledger, and workspace scoping — reachability makes existing systems reachable; it does not fork or duplicate their semantics.
- No Compozy-operated infrastructure: all traffic flows over the operator's machine and the operator's own provider accounts.
- The greenfield cut replaces today's exposure semantics outright (a non-local bind currently disables the entire API); the replacement is this feature's semantics, with no compatibility mode.
- Performance from the user's perspective: the remote UI remains usable on high-latency links with live streams degrading gracefully and recovering without data loss; public-ingress load must not starve local or overlay operation.
- Security requirements: fail-closed exposure, mandatory verification before ingress side effects, universal secret redaction, immediate revocation, and audit-grade attribution — as specified in Business Rules.
- Privacy: no new telemetry; posture data stays on the daemon.
- Platform coverage matches where the daemon ships today; platform-specific caveats (if any) are TechSpec concerns and must be stated, not silent.
- Naming note: SSH is named in this PRD because operator-held SSH access is itself the user-observable surface of that connection method (allowed exception, per the spec-authoring playbook).

## Non-Goals (Out of Scope)

- **Compozy-hosted infrastructure of any kind** — no relay service, no cloud-hosted daemons, no traffic through Compozy-operated servers (ADR-001). Revisiting this is a future product decision, unblocked by the provider seam.
- **Multi-user accounts, roles, or guest access** — the identity model is one operator, many devices (ADR-002). Teams are a separate future initiative.
- **Native mobile apps** — the responsive web UI over the overlay/public address is the mobile client.
- **Cross-machine Compozy Network peering** — agent-to-agent coordination across daemons is an evolution of the network subsystem, not of this feature; nothing here changes network semantics.
- **Webhook store-and-forward durability** — buffering, retrying, or replaying deliveries while the daemon is offline requires a persistent broker in the delivery path; explicitly out (documented limitation; sender-side redelivery is the mitigation).
- **Additional connectivity providers in the initial release** — exactly one bundled provider ships first; further providers (e.g., custom-domain public addresses) arrive through the provider seam afterward.
- **Managing operator-owned reverse proxies** — operators may still front a local-only daemon with their own infrastructure; this feature neither manages nor obstructs that setup.

## Architecture Decision Records

- [ADR-001: Remote access ships as first-party managed connectivity without Compozy-hosted infrastructure](adrs/adr-001.md) — managed overlay + public ingress in; hosted relay out.
- [ADR-002: Single-operator identity with per-device pairing and revocable device sessions](adrs/adr-002.md) — pairing over passwords; local access is the root of trust.
- [ADR-003: Exposure policy lives in core; connectivity mechanisms ship as provider extensions](adrs/adr-003.md) — policy/mechanism split; bundled first-party provider; trust-gated third parties.
- [ADR-004: Public address exposes surfaces per-surface, webhook ingress only by default](adrs/adr-004.md) — ingress-only default; consented opt-in for the operator surface.
- [ADR-005: Remote client methods — browser, remote CLI profiles, and SSH connection](adrs/adr-005.md) — three client paths, one pairing model.
- [ADR-006: Fail-closed exposure with agent-operable self-audit](adrs/adr-006.md) — unsafe state unrepresentable; audit as a first-class verb.

## Open Questions

- Which connectivity provider ships second (a custom-domain public-address vendor is the leading candidate), and what ordering the provider roadmap follows — parked for post-release planning; the seam makes this additive.
- Whether the SSH method should offer guided installation when Compozy is absent on the remote machine, instead of erroring with instructions — parked pending real usage feedback.
- Whether long-inactive device sessions should auto-expire by policy in addition to manual bulk cleanup — parked; the device list ships with manual management first.

## Research References

Competitor implementations studied for this PRD (implementers should read these alongside the TechSpec):

- Hermes — connection modes, fail-closed auth gate, two-leg connection testing, SSH lifecycle: `.resources/hermes/hermes_cli/web_server.py`, `.resources/hermes/apps/desktop/electron/connection-config.ts`, `.resources/hermes/apps/desktop/electron/ssh-connection.ts`, `.resources/hermes/apps/desktop/electron/remote-lifecycle.ts`, `.resources/hermes/website/docs/user-guide/desktop.md`.
- t3code — connection targets, pairing artifacts, managed endpoints, per-call authorization: `.resources/t3code/docs/internals/remote.md`, `.resources/t3code/packages/client-runtime/src/connection/model.ts`, `.resources/t3code/apps/server/src/startupAccess.ts`, `.resources/t3code/docs/internals/environment-auth.md`, `.resources/t3code/infra/relay/src/environments/ManagedEndpointProvider.ts`.
- OpenClaw — core exposure policy, audit command, the extension-fork anti-pattern: `.resources/openclaw/src/infra/tailscale.ts`, `.resources/openclaw/src/gateway/server-tailscale.ts`, `.resources/openclaw/src/security/audit-gateway-config.ts`, `.resources/openclaw/extensions/voice-call/src/tunnel.ts`.
- goclaw — embedded overlay node in a Go daemon: `.resources/goclaw/cmd/gateway_tsnet.go`.
