# User Stories: Remote Gateway

Canonical behavior catalog for the Remote Gateway feature. Companion to `_prd.md`; consumed by
`_techspec.md` (component mapping) and `_tests.md` (coverage matrix).

## Personas

- **Operator** — the single person who owns this Compozy installation. Runs the daemon on a home/office machine or a server, works from several personal devices (laptop, phone, work machine), and wants to reach their agent fleet from anywhere and let external services trigger work.
- **Agent** — an autonomous Compozy session (local or remote) that inspects, configures, operates, and repairs the gateway through structured command surfaces, without the web UI.
- **External service** — an internet service (e.g., GitHub) that delivers signed webhook events to the daemon's public ingress address. It follows its own delivery rules and never authenticates as the operator.
- **Provider developer** — an extension author who builds a connectivity provider (a mechanism that makes the daemon reachable) against the provider contract.
- **Unauthenticated visitor** — anyone who reaches an exposed address without a paired device session: a stranger, a scanner bot, or the operator on a not-yet-paired device.

## Story Index

| ID     | Feature Area          | Persona                 | Story                                                        |
| ------ | --------------------- | ----------------------- | ------------------------------------------------------------ |
| US-001 | Gateway status        | Operator                | See the full exposure posture at a glance                    |
| US-002 | Exposure modes        | Operator                | Enable private overlay access with nothing extra to install  |
| US-003 | Exposure modes        | Operator                | Unauthenticated exposure is impossible, with clear refusals  |
| US-004 | Exposure modes        | Operator                | Disable any exposure instantly and deterministically         |
| US-005 | Device pairing        | Operator                | Pair a new device with a one-time link/QR                    |
| US-006 | Device pairing        | Operator                | Manage the device list: rename, revoke, inspect              |
| US-007 | Device pairing        | Operator                | Clear experience on a revoked or expired device              |
| US-008 | Device pairing        | Agent                   | Inspect and manage devices through structured output         |
| US-009 | Private overlay       | Operator                | Use the full product from any paired device, anywhere        |
| US-010 | Private overlay       | Operator                | Overlay reachability is not authentication                   |
| US-011 | Public address        | Operator                | Obtain a stable public ingress address                       |
| US-012 | Public address        | Operator                | Opt the operator UI into public reach with explicit consent  |
| US-013 | Public address        | Unauthenticated visitor | Public probing yields nothing sensitive                      |
| US-014 | Webhook ingress       | Operator                | Get a shareable delivery URL for each webhook trigger        |
| US-015 | Webhook ingress       | External service        | Signed deliveries are verified and dispatched                |
| US-016 | Webhook ingress       | Operator                | A repository push starts a Loop end to end                   |
| US-017 | Webhook ingress       | Operator                | Bridges bind their inbound callbacks to the public address   |
| US-018 | Webhook ingress       | Operator                | Ingress verification is independent of operator auth         |
| US-019 | Remote CLI            | Operator                | Create a CLI connection profile by pairing                   |
| US-020 | Remote CLI            | Operator                | Operate a remote daemon with full command parity             |
| US-021 | Remote CLI            | Agent                   | Manage a remote daemon headlessly                            |
| US-022 | SSH connection        | Operator                | Connect over SSH: launch or reuse the daemon, work locally   |
| US-023 | SSH connection        | Operator                | The SSH method exposes nothing publicly                      |
| US-024 | Provider extensions   | Provider developer      | Ship a connectivity provider as an extension                 |
| US-025 | Provider extensions   | Operator                | Install a third-party provider under trust gating            |
| US-026 | Security & audit      | Operator, Agent         | Run the security self-audit                                  |
| US-027 | Security & audit      | Operator                | Secrets never leak across surfaces                           |
| US-028 | Security & audit      | Operator                | Remote actions carry device attribution                      |

## Gateway status

### US-001: See the full exposure posture at a glance

**As an** Operator, **I want** one place that shows how my daemon is currently reachable, **so that** I always know my exposure posture without reconstructing it from config.

Acceptance criteria:

- AC-1: Given any daemon state, when the operator opens the gateway status surface (UI panel or status command), then it shows: the active exposure mode(s), every advertised address, which surfaces (operator UI/API, webhook ingress) are reachable at each address, the paired-device count, and each connectivity provider's health.
- AC-2: Given a freshly installed daemon with no remote configuration, when status is read, then it reports local-only reachability and explicitly states that no remote surface is exposed.
- AC-3: Given the same daemon, when status is read through the UI and through the structured command, then both report the same facts.

Edge cases:

- EC-1: Daemon machine has no internet connectivity → status reports provider/public reachability as degraded with the cause, instead of showing stale addresses as live. (Interruption)
- EC-2: A provider reports an address the daemon cannot verify → status shows it as unverified/pending, never as active. (Invalid input)
- EC-3: Status read while an exposure change is mid-transition → reflects the transition state (enabling/disabling), not a stale terminal state. (Concurrency)
- EC-4: Zero devices, zero providers, zero webhooks → every list shows an explicit empty state, not an error. (Empty/missing)

## Exposure modes

### US-002: Enable private overlay access with nothing extra to install

**As an** Operator, **I want** to turn on private overlay access from the gateway settings using my own overlay-network account, **so that** my devices can reach the daemon from anywhere without me installing or configuring separate networking software on the daemon machine.

Acceptance criteria:

- AC-1: Given a daemon with only Compozy installed, when the operator enables the overlay in gateway settings (UI or CLI) and completes the provider's account authorization, then the daemon becomes reachable at a stable private address visible in status — with no other software installed by the operator.
- AC-2: Given the overlay is enabled, when the daemon restarts, then overlay reachability is restored automatically without re-authorization.
- AC-3: Given the enable flow, when the provider requires account authorization, then the operator completes it through an explicit approval step tied to their own provider account (never a Compozy-owned account).

Edge cases:

- EC-1: Provider authorization abandoned mid-flow → gateway remains in its previous state; retry starts cleanly. (Interruption)
- EC-2: Provider account lacks required permissions or quota → enabling fails with the provider's reason and a remediation hint; no partial exposure. (Limits)
- EC-3: Provider authorization expires or is revoked upstream later → status and audit surface the broken state; the daemon does not silently fall back to a different exposure path. (State transitions)
- EC-4: Enabling attempted twice concurrently (UI and CLI) → one succeeds, the other reports the in-progress/enabled state; no duplicate enrollment. (Concurrency, Repetition)

### US-003: Unauthenticated exposure is impossible, with clear refusals

**As an** Operator, **I want** the daemon to refuse any configuration that would expose it without authentication, **so that** no mistake of mine can produce an open daemon.

Acceptance criteria:

- AC-1: Given any path to non-local reachability (overlay enable, public address enable, direct non-local bind), when the authentication model is not active, then the transition is refused with a message naming the exact cause and fix.
- AC-2: Given the refusal, when the operator follows the named fix, then the same action succeeds.
- AC-3: There is no flag, environment toggle, or config value that bypasses the refusal.

Edge cases:

- EC-1: Config file hand-edited into a forbidden combination and the daemon restarted → daemon starts serving local-only and reports the refused remote config with the fix; it does not crash-loop and does not honor the unsafe part. (Invalid input)
- EC-2: Order inversion — operator tries to enable public UI before any pairing/auth is possible → refusal explains the required order. (Ordering)
- EC-3: A legacy/imported config from an older version requests exposure semantics that no longer exist → refused with guidance; no compatibility interpretation. (State transitions)

### US-004: Disable any exposure instantly and deterministically

**As an** Operator, **I want** to turn off any exposure with one action, **so that** I can cut remote reachability immediately when something feels wrong.

Acceptance criteria:

- AC-1: Given any active exposure, when the operator disables it (UI or CLI), then the corresponding addresses stop reaching the daemon immediately and status reflects it.
- AC-2: Given a full teardown ("back to local-only"), when invoked, then all remote reachability ends while local access continues working unchanged.
- AC-3: Given a disabled surface, when the daemon restarts, then the surface stays disabled — restart never re-exposes anything the operator turned off.

Edge cases:

- EC-1: Disable while remote device sessions are active → their connections terminate; the local machine's access is unaffected. (Interruption)
- EC-2: Disable while a webhook delivery is in flight → the in-flight request completes or fails cleanly; no new deliveries are accepted. (Concurrency)
- EC-3: Daemon crashes instead of clean shutdown → on restart, exposure state matches the operator's last configured intent; nothing turned-off comes back on. (Interruption)
- EC-4: Disable invoked twice → second invocation is a no-op with a clear "already disabled" result. (Repetition)

## Device pairing

### US-005: Pair a new device with a one-time link/QR

**As an** Operator, **I want** to admit a new device by scanning a code or opening a link minted from a trusted surface, **so that** joining is quick and there is never a password to type or store.

Acceptance criteria:

- AC-1: Given local access to the daemon (machine UI or CLI), when the operator mints a pairing artifact, then it is presented as both a scannable code and a copyable link/text code, is single-use, and expires within minutes.
- AC-2: Given a valid unexpired artifact, when the new device completes pairing, then it gains a named device session and appears immediately in the device list with its origin.
- AC-3: Given a completed pairing, when the same artifact is presented again, then it is rejected.

Edge cases:

- EC-1: Expired artifact used → rejected with a "mint a new one" hint; no session created. (State transitions)
- EC-2: Malformed or truncated artifact value presented → rejected generically; the response does not reveal whether an artifact exists. (Invalid input)
- EC-3: Artifact intercepted and used by someone else first → the interloper's device appears in the device list like any pairing (visible origin, immediately revocable); the operator's own attempt then fails because the artifact is spent — a visible anomaly rather than a silent one. (Permissions)
- EC-4: Two devices race the same artifact → exactly one wins; the other gets the spent-artifact rejection. (Concurrency)
- EC-5: Many artifacts minted and abandoned → pending artifacts are bounded and expire; minting more than the bound replaces or refuses oldest-first with a clear message. (Limits, Scale)
- EC-6: Pairing attempted while the daemon has zero exposure enabled → pairing from local access still works (it is the bootstrap path); remote pairing completion requires a reachable address and says so. (Ordering, Empty/missing)

### US-006: Manage the device list: rename, revoke, inspect

**As an** Operator, **I want** a device inventory with rename and revoke, **so that** I can recognize every session that can reach my daemon and cut any of them instantly.

Acceptance criteria:

- AC-1: Given paired devices, when the operator opens the device list, then each entry shows name, pairing origin, creation time, and last activity.
- AC-2: Given a device entry, when the operator revokes it, then that device's access ends immediately — including live streams — and the entry shows as revoked.
- AC-3: Given a device entry, when renamed, then the new name appears consistently across UI, status, and audit output.

Edge cases:

- EC-1: Revoking the device currently being used to perform the revocation → allowed with an explicit warning; the session ends immediately after. (State transitions)
- EC-2: Revoking the last remote device → remote access ends entirely; local machine access is unaffected; re-pairing from local access remains possible. (Empty/missing)
- EC-3: Two surfaces revoke the same device concurrently → both succeed idempotently; one audit event records the revocation. (Concurrency, Repetition)
- EC-4: Device list at scale (dozens of stale devices) → list remains usable (ordering by last activity) and bulk cleanup of long-inactive sessions is possible. (Scale)

### US-007: Clear experience on a revoked or expired device

**As an** Operator, **I want** a revoked or expired device to see an unambiguous "access ended" state, **so that** I am never confused about why a device stopped working.

Acceptance criteria:

- AC-1: Given a revoked device, when it next interacts (or has a live view open), then it lands on a screen/output stating access was revoked and how to re-pair — with no residual data visible.
- AC-2: Given an expired session (if sessions carry expiry), when the device returns, then the renewal or re-pair path is presented without data exposure.

Edge cases:

- EC-1: Revocation happens mid-action (e.g., while submitting a prompt) → the action fails with the revoked outcome; no partial write is silently kept. (Interruption)
- EC-2: A revoked device retries in a tight loop → rejections are rate-limited and logged once per burst, not once per attempt. (Repetition, Limits)
- EC-3: A revoked device attempts the pairing endpoint with its old credentials → treated as unauthenticated; only a fresh pairing artifact admits it. (Permissions)

### US-008: Inspect and manage devices through structured output

**As an** Agent, **I want** device inventory and revocation available as structured commands, **so that** I can audit and repair access without the web UI.

Acceptance criteria:

- AC-1: Given any device state, when the agent lists devices via the structured command surface, then output is machine-readable, complete, and consistent with the UI view.
- AC-2: Given a revocation command with a device identifier, when executed, then the result is deterministic (revoked / already-revoked / not-found) with distinct, stable error identities.

Edge cases:

- EC-1: Unknown device identifier → deterministic not-found error, not a generic failure. (Invalid input)
- EC-2: Empty device list → valid empty structured result. (Empty/missing)
- EC-3: Agent runs the same revocation twice → second call reports already-revoked; exit status distinguishes "changed" from "no-op". (Repetition)

## Private overlay

### US-009: Use the full product from any paired device, anywhere

**As an** Operator, **I want** the complete Compozy experience from my phone or laptop over the private overlay, **so that** being away from the machine never limits what I can do.

Acceptance criteria:

- AC-1: Given the overlay is enabled and the device is paired, when the operator opens the daemon's private address in a browser, then the full web UI works: sessions, live streams, approvals, settings — the same feature set as local use.
- AC-2: Given a live session view over the overlay, when events occur, then streaming updates arrive continuously (no degraded polling mode).
- AC-3: Given the operator's device moves networks (cellular → wifi), when connectivity resumes, then the session reconnects without re-pairing.

Edge cases:

- EC-1: Daemon machine asleep/offline → the client fails with a reachability error attributable to the daemon being offline; no misleading auth error. (Interruption)
- EC-2: Long-running work started from a remote device, device disconnects → work continues on the daemon; reconnecting shows its progress (detached execution lifetime). (Interruption)
- EC-3: Two paired devices operate the same workspace simultaneously → both see consistent live state; no device-exclusive locks appear. (Concurrency)
- EC-4: Very high-latency link → UI remains functional; streams degrade gracefully without data loss on reconnect replay. (Limits)

### US-010: Overlay reachability is not authentication

**As an** Operator, **I want** overlay-reachable-but-unpaired devices to hit the pairing gate, **so that** anyone with access to my overlay network still cannot operate my daemon.

Acceptance criteria:

- AC-1: Given a device that can reach the private address but holds no device session, when it opens the UI, then it sees only the pairing/access gate — no workspace names, session data, or daemon details.
- AC-2: Given the same device, when it calls any operator API path, then it receives a deterministic unauthenticated rejection.

Edge cases:

- EC-1: A second person is admitted to the operator's overlay network (shared tailnet) → they are an Unauthenticated visitor at the gate; nothing more. (Permissions)
- EC-2: Structured probing of API paths from an unpaired overlay device → uniform rejections that do not enumerate which paths exist. (Invalid input)

## Public address

### US-011: Obtain a stable public ingress address

**As an** Operator, **I want** a stable public web address for my daemon's ingress, **so that** I can paste it into external services once and it keeps working.

Acceptance criteria:

- AC-1: Given the public address is enabled through a provider, when the operator views gateway status, then the exact public base address is shown and copyable in UI and CLI.
- AC-2: Given a daemon restart, when the public address is re-established, then it is the same address (stability across restarts is the contract).
- AC-3: Given the public address is enabled with defaults, when any non-ingress path is requested on it, then the request is refused as unreachable (webhook-only default per ADR-004).

Edge cases:

- EC-1: Provider cannot provide a stable address (quota, plan limits) → enabling fails with the provider's reason; no unstable address is silently accepted. (Limits)
- EC-2: Provider changes the address out from under us → status and audit flag the change prominently; dependent consumers (webhook URLs, bridges) are marked as needing re-confirmation. (State transitions)
- EC-3: Public enable attempted while overlay is disabled → allowed (independent surfaces); status makes the resulting posture explicit. (Ordering)

### US-012: Opt the operator UI into public reach with explicit consent

**As an** Operator, **I want** to consciously enable the full UI on my public address, **so that** I can work from a device that cannot join my overlay — accepting the trade-off explicitly.

Acceptance criteria:

- AC-1: Given a public address serving ingress only, when the operator enables the public UI surface, then a consent step states exactly what becomes publicly reachable before it takes effect; consent is re-required every time the surface is turned on.
- AC-2: Given public UI enabled, when an unpaired client opens the address, then it sees only the pairing/access gate; a paired device gets the full UI.
- AC-3: Given the operator disables the public UI surface, when a public client next interacts, then the UI is unreachable again immediately; webhook ingress remains unaffected.

Edge cases:

- EC-1: Consent flow abandoned → surface stays off. (Interruption)
- EC-2: Public UI enabled, then all devices revoked → the gate remains; re-pairing requires local access (the public gate never mints artifacts). (State transitions, Permissions)
- EC-3: Repeated toggling on/off → each on requires consent; each off is immediate; no residual reachability between cycles. (Repetition)

### US-013: Public probing yields nothing sensitive

**As an** Unauthenticated visitor (from the operator's threat perspective), **I want** every public request without credentials to hit a uniform wall, **so that** scanners and strangers learn nothing about the daemon.

Acceptance criteria:

- AC-1: Given any public surface configuration, when unauthenticated requests probe arbitrary paths, then responses reveal no workspace data, version details beyond the minimum, or path enumeration signal.
- AC-2: Given repeated failed access attempts from one source, when the pattern continues, then responses are rate-limited and the pattern is visible in audit output.

Edge cases:

- EC-1: Probe floods at scale → the daemon's local and overlay operation remains responsive (ingress load does not starve operator surfaces). (Scale, Limits)
- EC-2: Malformed requests (oversized bodies, invalid encodings) on public paths → rejected within bounds without resource exhaustion. (Invalid input)

## Webhook ingress

### US-014: Get a shareable delivery URL for each webhook trigger

**As an** Operator, **I want** every webhook trigger to show its exact public delivery URL, **so that** configuring GitHub (or any sender) is copy-paste.

Acceptance criteria:

- AC-1: Given a webhook trigger and an enabled public address, when the operator views the trigger, then its full public delivery URL is displayed and copyable in UI and CLI.
- AC-2: Given the public address is disabled, when the operator views the trigger, then the URL area states that public delivery is off and links the enable path (no dead URL shown as live).
- AC-3: Given a new webhook trigger is created, when the operator completes creation, then its verification secret setup and delivery URL are presented together as one flow.

Edge cases:

- EC-1: Trigger exists from before the gateway feature → it gains a delivery URL like any other once public ingress is on; no re-creation needed. (State transitions)
- EC-2: Public address changes → all trigger URL displays update; the operator is prompted to update external senders. (State transitions)
- EC-3: Zero webhook triggers → ingress-focused surfaces show an explicit empty state with a create path. (Empty/missing)

### US-015: Signed deliveries are verified and dispatched

**As an** External service, **I want** my signed webhook deliveries accepted exactly when they verify, **so that** the operator's automation runs on real events and nothing else.

Acceptance criteria:

- AC-1: Given a delivery signed with the trigger's secret and a fresh timestamp, when it arrives at the public delivery URL, then it is accepted and dispatched to the automation trigger (including starting a Loop when so configured).
- AC-2: Given a delivery with an invalid signature, when it arrives, then it is rejected with no side effects and the rejection is observable to the operator.
- AC-3: Given a delivery replayed with an already-seen delivery identity, when it arrives, then it is deduplicated (no second dispatch).

Edge cases:

- EC-1: Stale timestamp outside the freshness window → rejected; observable reason distinguishes staleness from bad signature. (Invalid input)
- EC-2: Payload above the size bound → rejected before processing. (Limits)
- EC-3: Burst of valid deliveries → accepted within rate bounds; beyond bounds, deliveries are rejected with a retry-appropriate outcome rather than silently dropped. (Scale, Limits)
- EC-4: Delivery for a disabled or deleted trigger → rejected deterministically; no orphaned side effects. (State transitions)
- EC-5: Two identical valid deliveries race → exactly one dispatch (delivery-identity claim is atomic). (Concurrency)

### US-016: A repository push starts a Loop end to end

**As an** Operator, **I want** to configure "on push, run my Loop" and see it work minutes later, **so that** the motivating scenario is a first-class journey, not an integration project.

Acceptance criteria:

- AC-1: Given an enabled public address, a webhook trigger bound to a Loop, and the delivery URL configured in the repository, when a push occurs, then a Loop run starts with the delivery payload available to it, observable in the runs surface.
- AC-2: Given the run started, when the operator inspects it, then its origin is attributed to the webhook trigger (which trigger, which delivery).
- AC-3: Given the sender's delivery log (e.g., GitHub's), when the delivery succeeds, then the sender records a success outcome (round-trip health is verifiable from the sender side).

Edge cases:

- EC-1: Daemon offline at delivery time → the sender records a failed delivery; documentation states the durability boundary (no store-and-forward in this feature) and the sender-side redelivery path. (Interruption)
- EC-2: Loop start refused (e.g., concurrency budget) → the delivery is recorded with the refusal reason; the operator can see why no run started. (Limits)
- EC-3: Trigger misconfigured (secret mismatch between sender and trigger) → every delivery rejects with a signature failure the operator can diagnose from the observable rejection log. (Invalid input, Ordering)

### US-017: Bridges bind their inbound callbacks to the public address

**As an** Operator, **I want** bridge instances to use the gateway's public address for their inbound callbacks, **so that** I never build a reverse proxy per bridge again.

Acceptance criteria:

- AC-1: Given an enabled public address and a bridge instance requiring an inbound callback URL, when the operator configures the bridge, then they can bind it to the gateway's public address with a per-instance confirmation instead of entering external infrastructure.
- AC-2: Given a bound bridge, when its platform delivers a callback to the public address, then it reaches the bridge's verification and processing exactly as it would through a manual proxy.
- AC-3: Given a bound bridge, when the operator views bridge health, then ingress reachability is part of the health picture.

Edge cases:

- EC-1: Public address disabled while bridges are bound → affected bridges surface broken-ingress health with remediation; they do not silently queue forever. (State transitions)
- EC-2: Public address changes → bound bridges are flagged for re-confirmation and platform-side URL updates. (State transitions)
- EC-3: Operator keeps an external proxy for one bridge while binding another to the gateway → both configurations coexist; binding is per instance. (Ordering)

### US-018: Ingress verification is independent of operator auth

**As an** Operator, **I want** webhook verification and operator authentication to be entirely separate, **so that** neither can weaken the other.

Acceptance criteria:

- AC-1: Given a valid operator device session, when it posts an unsigned request to a delivery URL, then the request is rejected — operator identity never substitutes for delivery verification.
- AC-2: Given a valid signed delivery, when it arrives, then it is accepted regardless of any operator-session state — and it can do nothing except dispatch its trigger (no operator-surface access).
- AC-3: Given secrets management, when a trigger's verification secret is rotated, then old-secret deliveries fail from that point and the sender-side update path is documented.

Edge cases:

- EC-1: A delivery URL is pasted into a browser (plain visit) → a method-appropriate rejection with no daemon detail. (Invalid input)
- EC-2: Sender confuses the delivery-verification secret with a pairing artifact or other credential → distinct labeled concepts in UI/docs; the failure mode names which credential was wrong in operator-visible logs (never in the public response). (Invalid input, Permissions)

## Remote CLI

### US-019: Create a CLI connection profile by pairing

**As an** Operator, **I want** to pair a CLI on another machine as a named connection profile, **so that** my terminals hold their own revocable access to the remote daemon.

Acceptance criteria:

- AC-1: Given a pairing artifact minted on the daemon side, when the operator completes pairing from the CLI on another machine, then a named connection profile exists there, appears in the daemon's device list as its own entry, and is individually revocable.
- AC-2: Given multiple profiles (e.g., home daemon, office daemon), when the operator lists profiles, then each shows its target and pairing state.

Edge cases:

- EC-1: Pairing from a CLI that cannot reach the daemon's address → fails with a reachability error distinct from an artifact error. (Ordering)
- EC-2: A passphrase-protected profile export copied to another machine and imported there → treated as the same device identity; the operator can see activity and revoke it — documented as "protect profile bundles like keys". Copying `config.toml` alone never transfers the credential. (Permissions)
- EC-3: Creating a profile with a name that already exists → deterministic conflict with rename/overwrite choice. (Invalid input, Repetition)

### US-020: Operate a remote daemon with full command parity

**As an** Operator, **I want** every CLI command to work against a selected remote profile exactly as locally, **so that** remote operation is not a second-class dialect.

Acceptance criteria:

- AC-1: Given a valid profile, when the operator runs any CLI command targeted at it, then behavior, structured output, and streaming match local use, and the output/prompt context makes the remote target unmistakable.
- AC-2: Given a command that starts long-running work remotely, when the CLI disconnects mid-run, then the work continues on the daemon (detached lifetime) and a later invocation can observe its progress.
- AC-3: Given a revoked or expired profile, when any command runs, then a deterministic auth error explains re-pairing — distinct from network unreachability.

Edge cases:

- EC-1: Daemon unreachable → network error with the target named; no hang without feedback. (Interruption)
- EC-2: Mid-stream connection loss → the CLI reports the interruption and exits with a distinct outcome; no silent partial success. (Interruption)
- EC-3: A command that is inherently local-machine-scoped (if any exists) → refuses remotely with a deterministic "local-only" error rather than acting on the wrong machine. (Permissions, Invalid input)
- EC-4: Two terminals operate the same remote daemon concurrently → both work; results are consistent with two local terminals. (Concurrency)

### US-021: Manage a remote daemon headlessly

**As an** Agent, **I want** to operate a remote daemon through the same structured CLI surfaces, **so that** fleet chores (audit, config checks, repair) can be delegated to me.

Acceptance criteria:

- AC-1: Given a configured profile, when an agent runs structured commands against it, then outputs are machine-readable and errors deterministic — sufficient to script decisions without screen scraping.
- AC-2: Given an auth failure, when the agent encounters it, then the error identity is stable and distinguishable from transport failures, so the agent can decide "re-pair needed" vs "retry later".

Edge cases:

- EC-1: Agent operating with a revoked profile in a loop → stable repeated error; no lockout of unrelated profiles. (Repetition)
- EC-2: Structured output consumed while the connection degrades → output remains valid (complete records or explicit truncation error). (Interruption)

## SSH connection

### US-022: Connect over SSH: launch or reuse the daemon, work locally

**As an** Operator, **I want** to point Compozy at a machine I can SSH into and have it bring the daemon up and make it available locally, **so that** any server I already control becomes usable without configuring networking or providers.

Acceptance criteria:

- AC-1: Given a remote machine with Compozy installed and working SSH access (existing keys/config), when the operator runs the SSH connect command with the host, then Compozy launches the daemon there (or reuses a running one it can verify as compatible), establishes the local access path, and local UI/CLI reach it for the duration of the connection.
- AC-2: Given the connection is active, when the operator ends it, then only what the connection started is stopped: a daemon launched by the connection is shut down (or per the operator's choice left running); a pre-existing daemon is left untouched.
- AC-3: Given SSH authentication fails, when connecting, then the failure surfaces before any remote state changes, with the SSH-level reason.

Edge cases:

- EC-1: Compozy not installed on the remote machine → deterministic error naming the installation step; no partial bootstrap left behind. (Empty/missing)
- EC-2: Remote daemon version incompatible with the local client → refused with both versions named; no silent protocol degradation. (Invalid input, State transitions)
- EC-3: SSH connection drops mid-session → local access pauses with a reconnecting state; the remote daemon and its running work continue; reconnection resumes without re-launch. (Interruption)
- EC-4: Two SSH connections to the same remote machine (two laptops) → both reuse the same daemon safely or the second gets a clear busy/ownership outcome — never two daemons fighting over one home directory. (Concurrency)
- EC-5: Host key changed on the remote → connection refused with the standard trust warning; never auto-accepted. (Permissions)
- EC-6: Remote machine's Compozy home differs from default → connect honors the remote configuration; profile records what it connected to. (Ordering)

### US-023: The SSH method exposes nothing publicly

**As an** Operator, **I want** the SSH path to leave zero network exposure on both machines, **so that** it is always the most conservative option.

Acceptance criteria:

- AC-1: Given an active SSH connection, when either machine's network posture is inspected, then no new surface is reachable beyond each machine's local scope (the daemon stays local-only on the remote; the local access path is local-only on the client machine).
- AC-2: Given the SSH connection ends, when posture is inspected again, then everything the connection created is gone.

Edge cases:

- EC-1: Client machine is on a hostile LAN → the local access path is not reachable by other LAN hosts. (Permissions)
- EC-2: Remote daemon separately has gateway exposure configured by its own operator settings → the SSH method neither enables nor disables it; posture changes only through explicit gateway settings. (State transitions)

## Provider extensions

### US-024: Ship a connectivity provider as an extension

**As a** Provider developer, **I want** a stable provider contract for connectivity mechanisms, **so that** I can add a new way to make daemons reachable without touching the runtime.

Acceptance criteria:

- AC-1: Given the documented provider contract, when a provider extension declares the connectivity provide surface and implements its lifecycle (establish, report endpoints/health, teardown), then the daemon supervises it and its endpoints appear in gateway status like the bundled provider's.
- AC-2: Given a provider reports endpoints, when the daemon advertises them, then advertisement happens only after core-side verification — a provider cannot force an address into "active".
- AC-3: Given a provider crashes, when the daemon detects it, then affected reachability degrades to a safe state (never half-exposed), the failure is visible in status/audit, and supervised restart applies.

Edge cases:

- EC-1: Provider reports malformed endpoint data → rejected at validation; provider marked unhealthy with the reason. (Invalid input)
- EC-2: Provider hangs during teardown → bounded teardown; the daemon reports the degraded provider rather than hanging shutdown. (Interruption, Limits)
- EC-3: Two providers claim overlapping roles (both provide a public address) → the operator chooses which is active per role; no silent double-exposure. (Concurrency, Ordering)

### US-025: Install a third-party provider under trust gating

**As an** Operator, **I want** third-party connectivity providers to require my explicit, informed consent, **so that** nothing can quietly change how my daemon is reached.

Acceptance criteria:

- AC-1: Given a third-party provider extension, when installing/enabling it, then the consent step states that this extension controls daemon reachability, its install source tier, and what it will manage; without consent nothing activates.
- AC-2: Given install-source policy, when a provider comes from an untrusted source, then enabling is blocked by default policy, with the policy surface documented.
- AC-3: Given an enabled third-party provider, when the operator disables it, then its reachability contribution is removed immediately.

Edge cases:

- EC-1: Provider extension updated to a new version → re-consent is required when its declared control surface expands. (State transitions)
- EC-2: Provider tries to activate outside its declared role → refused by the contract boundary; visible in audit. (Permissions)

## Security & audit

### US-026: Run the security self-audit

**As an** Operator or Agent, **I want** an on-demand audit of the daemon's exposure posture, **so that** "am I safe?" has a first-class, actionable answer.

Acceptance criteria:

- AC-1: Given any daemon state, when the audit runs (UI or structured command), then it reports: exposure modes and addresses, per-surface public reachability, auth posture, device inventory highlights (stale devices, recent pairings), provider health, webhook trigger verification posture, and findings ranked by severity with concrete remediation.
- AC-2: Given a finding, when the operator or agent applies the named remediation, then a re-run clears that finding.
- AC-3: Given a fully local-only daemon, when audited, then the audit still runs and reports the minimal posture (audit is not remote-only).
- AC-4: Given agent consumption, when the audit runs in structured mode, then findings carry stable identities and machine-readable severity/remediation fields.

Edge cases:

- EC-1: Audit during a provider outage → reports the degraded provider as a finding, not an audit failure. (Interruption)
- EC-2: No findings → explicit "no findings" result, distinguishable from "audit did not run". (Empty/missing)
- EC-3: Audit invoked repeatedly in a loop by an agent → cheap and side-effect-free; results consistent absent state changes. (Repetition)

### US-027: Secrets never leak across surfaces

**As an** Operator, **I want** every gateway-related secret redacted everywhere it might appear, **so that** logs, status, and diagnostics are safe to share.

Acceptance criteria:

- AC-1: Given pairing artifacts, provider credentials, and webhook verification secrets, when any of them would appear in logs, status output, audit reports, events, or error messages, then only redacted or fingerprint forms appear.
- AC-2: Given a diagnostics/export flow, when produced, then it is redacted by construction (safe to attach to an issue).

Edge cases:

- EC-1: Secret embedded in a URL a user pastes into the wrong field → rendering surfaces redact recognized secret shapes on display. (Invalid input)
- EC-2: Provider subprocess output contains its own credential material → supervised output is redacted before logging. (Invalid input)

### US-028: Remote actions carry device attribution

**As an** Operator, **I want** actions performed remotely to be attributable to the device session that performed them, **so that** the operational ledger answers "which device did this?".

Acceptance criteria:

- AC-1: Given a remote device performs a state-changing action, when the operator inspects the operational events, then the event carries the device session identity alongside existing correlation fields.
- AC-2: Given a revoked device's past actions, when inspected later, then attribution remains intact (revocation does not erase history).

Edge cases:

- EC-1: Actions from the local machine → attributed as local, so remote vs local is always distinguishable. (Ordering)
- EC-2: Attribution at scale (thousands of events) → filtering events by device is possible in the same way as by other correlation fields. (Scale)
