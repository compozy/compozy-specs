---
status: completed
title: Gateway operator surface and documentation
type: frontend
complexity: high
---

# Task 7: Gateway operator surface and documentation

## Overview

Gives the operator a place to see and drive everything the previous tasks built: a gateway settings surface with exposure modes, the device inventory, pairing, the public-UI consent step, and the audit panel — plus migration of existing live-stream consumers to ticket-based authentication so the web app works over a remote session. It ships with the operator runbook and threat-model documentation, written by whoever just built the surface.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- The surface MUST present named exposure modes with plain-language risk framing, never a bare toggle.
- The public operator-UI consent step MUST state concretely what becomes publicly reachable, and consent MUST be re-required on every enable.
- Reachability shown in the UI MUST reflect verified runtime state; a provider claim that failed verification MUST render as pending or degraded, never as active.
- Every scannable pairing code MUST have a copyable text equivalent for screen readers, remote terminals, and devices without cameras.
- The device list MUST show name, origin, and last activity, and MUST support rename and immediate revoke.
- A revoked device MUST land on an explicit access-ended state with no residual data visible.
- Live-stream consumers MUST acquire a fresh single-use ticket per connection and per reconnect when the session is a remote gateway session.
- The surface MUST NOT render controls or metrics the runtime does not support; on conflict, daemon truth wins.
- Reuse `@compozy/ui` primitives; do not shadow an exported primitive name, and give domain variants domain-prefixed names.
- No visual reference exists for this surface, so the design system tokens and the copy specification govern; no Visual Contract section applies.
- Documentation MUST include an operator runbook for each flow and a threat-model page stating what each tier exposes and what the fail-closed guarantees are.
- Documentation MUST state the delivery durability boundary wherever a delivery URL is displayed.
</requirements>

## Subtasks

- [x] 7.1 Build the gateway settings surface with exposure modes, per-tier status, and advertised addresses
- [x] 7.2 Build the pairing flow with a scannable code and its copyable text equivalent, plus expiry and re-mint affordances
- [x] 7.3 Build the device inventory with rename, revoke, origin, and last activity
- [x] 7.4 Build the public operator-UI consent step with concrete disclosure and per-enable re-consent
- [x] 7.5 Build the audit panel rendering ranked findings with their remediation
- [x] 7.6 Build the access-ended state for revoked or expired device sessions
- [x] 7.7 Migrate every live-stream consumer to ticket-based authentication with fresh tickets on reconnect, preferring the shared SSE source over per-hook changes, and cover the WebSocket upgrade path too
- [x] 7.8 Surface trigger delivery URLs and bridge ingress health, including honest off-state messaging
- [x] 7.9 Write the operator runbook covering overlay enablement, pairing, public ingress, remote CLI, and SSH
- [x] 7.10 Write the threat-model page describing each tier's exposure and the fail-closed guarantees
- [x] 7.11 Ship the configuration reference for the gateway keys, regenerate the CLI reference, and register the new pages in their directory ordering files

## Implementation Details

Follow the TechSpec sections *User Experience* (via the PRD), *Credential Ownership and Stream Flow*, and *Config Lifecycle*. The web client currently derives its API base from the page origin, which already works for a daemon-served remote surface; the change needed is ticket acquisition for streams, not a backend picker. Activate the design and UI-craft skills before authoring components, and check the shared primitive inventory before creating any new component.

### Relevant Files

- `web/src/lib/api-client.ts` — same-origin API base; streams need ticket acquisition layered on (`web/src/lib/api-contract.ts` for contract helpers)
- `web/src/systems/session/hooks/session-stream-source.ts` — the shared SSE source; the single highest-leverage place to add ticket acquisition
- Live-stream consumers that must acquire a fresh ticket per connection and per reconnect: `web/src/systems/session/hooks/use-session-catalog-streams.ts`, `web/src/systems/session/hooks/use-session-live-tail.ts`, `web/src/systems/session/hooks/use-merged-session-runtime-transcript.ts`, `web/src/systems/session/components/session-chat-runtime-provider.tsx`, `web/src/systems/tasks/hooks/use-task-stream.ts`, `web/src/systems/loops/hooks/use-loop-stream.ts`, `web/src/systems/bridges/hooks/use-bridge-health-stream.ts`, `web/src/systems/extensions/hooks/use-extension-logs.ts`, `web/src/systems/dashboard/hooks/use-home-live.ts`, `web/src/systems/network/hooks/use-task-run-conversation.ts`, `web/src/systems/os/hooks/use-window-manager-stream.ts` (WebSocket, not SSE — needs the same ticket on upgrade)
- `web/src/routes/_app/settings/-network-settings-page.tsx` — the settings page pattern to copy for the gateway panel
- `web/src/routes/_app/settings/extensions.tsx` — the route-entry pattern; register the new page in `web/src/routes/_app/settings.ts`
- `web/src/systems/settings/` — adapters, components, hooks, stores, and types the gateway panel plugs into
- `web/src/systems/os/components/` — existing OS shell components and modal patterns for the pairing dialog
- `packages/ui/src/index.ts` — the primitive inventory to reuse before authoring anything new
- `packages/ui/src/tokens.css` — canonical token source for color, type, spacing, radius, and motion
- `DESIGN.md` — generated token specification and grammar
- `COPY.md` — product-language specification governing all visible copy
- `web/src/generated/compozy-openapi.d.ts` — generated types for the gateway endpoints
- `packages/site/content/docs/operations/` — where the runbook and threat-model pages land; **`meta.json` in that directory controls page order and must list the new pages**
- `packages/site/content/docs/configuration/config-toml.mdx` — gateway key reference (plus `lifecycle-matrix.mdx` for restart-versus-live classification)
- `packages/site/content/docs/automation/webhooks.mdx` — delivery URL and durability boundary
- `packages/site/content/docs/bridges/setup.mdx` — bridge binding replaces manual proxy guidance
- `packages/site/content/docs/cli/` — **generated**, not hand-written: produced by `magefiles/codegen_cli_docs.go` via `make codegen` and drift-checked by `scripts/check-cli-docs.sh`; a per-group directory with its own `meta.json` appears for the new command group

### Dependent Files

- `web/src/systems/os/apps/` — any app hosting live streams inherits ticket acquisition
- `packages/site/content/docs/cli/` — generated CLI reference for gateway verbs
- `web/src/systems/settings/__tests__/` — settings surface tests

### Related ADRs

- [ADR-004: Public tier surface set](adrs/adr-004.md) — what the consent step must disclose
- [ADR-002: Single-operator identity with per-device pairing](adrs/adr-002.md) — device inventory and pairing semantics
- [ADR-006: Fail-closed exposure with agent-operable self-audit](adrs/adr-006.md) — audit panel content and discoverability
- [ADR-008: Opaque revocable credentials](adrs/adr-008.md) — cookie-based browser sessions and stream tickets

## Deliverables

- Gateway settings surface with exposure modes, addresses, and per-tier status
- Pairing flow with scannable and copyable artifacts, device inventory with rename and revoke, and the access-ended state
- Public operator-UI consent step with concrete disclosure and per-enable re-consent
- Audit panel rendering ranked findings with remediation
- Live-stream consumers migrated to ticket-based authentication with fresh tickets on reconnect
- Operator runbook and threat-model documentation, plus configuration and CLI reference updates
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] E2E-001 — enable the private tier, mint a pairing, redeem it from a second browser context, load the full UI with a live stream, and confirm an unpaired context sees only the gate
- [x] E2E-005 — consent to the public operator UI, reach it from a paired device, revoke that device and observe the access-ended state, then disable exposure and confirm local-only
- [x] E2E-006 — run the self-audit, observe a finding, apply its remediation, and confirm the re-run reports it cleared

## Success Criteria

- Every assigned test case implemented and passing
- Every pairing artifact is obtainable without a camera
- The audit panel renders findings whose remediation text is actionable without reading the source
- `make bun-lint` and `make bun-typecheck` pass with zero findings
- No control or metric appears that the runtime does not support
- The threat-model page states, per tier, exactly what is reachable and what fail-closed guarantees hold
