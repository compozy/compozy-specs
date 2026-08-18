---
status: completed
title: "Public ingress: webhook and bridge reachability"
type: backend
complexity: high
---

# Task 4: Public ingress: webhook and bridge reachability

## Overview

Makes the public tier useful: every webhook trigger projects a copy-paste delivery URL, bridges bind their inbound callbacks to the gateway address instead of operator-built proxies, and both are anchored by workspace-authorized binding records that invalidate when the public address changes. This is the slice that turns "a repository push starts a Loop on my machine" from a networking project into a product journey.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- Existing webhook verification (signature, timestamp freshness, delivery-identity replay) MUST remain unchanged; this task supplies reachability and addressing, not new verification.
- Ingress verification MUST stay independent of operator authentication: an operator session MUST NOT substitute for a signature, and a valid delivery MUST grant nothing beyond dispatching its own trigger.
- `gateway_ingress_bindings` MUST store the subject's server-resolved `scope_kind`/`workspace_id`; client-supplied scope MUST never be trusted.
- Agent callers MUST be authorized through the canonical workspace boundary before any binding mutation; a cross-workspace bind or unbind MUST be denied deterministically.
- A binding MUST record the `endpoint_generation` it was confirmed against, so a public-address change invalidates confirmations instead of silently redirecting a sender.
- Deleting a webhook trigger or bridge instance MUST remove its binding in the same transaction; orphaned bindings MUST be swept on reconcile and treated as absent by reads.
- A trigger's delivery URL surface MUST report honestly when public ingress is off — never a dead URL presented as live.
- Bridge binding MUST be per instance with explicit confirmation, and bridge health MUST include ingress reachability; an externally proxied bridge MUST keep working untouched.
- Delivery reachability is bounded by daemon uptime — there is no store-and-forward — and that limitation MUST be documented wherever a delivery URL is displayed.
- Route, contract, OpenAPI, TypeScript, and event registration for this surface MUST co-ship in this task.
</requirements>

## Subtasks

- [x] 4.1 Add the ingress-binding table with scope, workspace ownership, and endpoint generation via declarative fragment plus appended migration
- [x] 4.2 Project a public delivery URL per webhook trigger from the verified public endpoint and the existing endpoint formatter
- [x] 4.3 Implement honest URL surfacing when public ingress is off, including the enable path and the empty-trigger state
- [x] 4.4 Implement binding create/delete with server-side subject resolution and canonical workspace authorization
- [x] 4.5 Implement endpoint-generation invalidation so an address change flags dependents for re-confirmation
- [x] 4.6 Implement transactional binding cleanup on subject deletion plus reconcile-time orphan sweeping
- [x] 4.7 Wire bridge instances to bind their inbound callback to the public address and surface ingress reachability in bridge health
- [x] 4.8 Register the ingress management routes on the private tier and the UDS subset with contract types and generated artifacts
- [x] 4.9 Register ingress-related canonical events with redaction mapping

## Implementation Details

Follow the TechSpec sections *Data Models* (`gateway_ingress_bindings`), *API Endpoints*, *Integration Points*, and *Safety Invariant 16*. The webhook verification chain already exists and is not modified — the endpoint parser/formatter, freshness window, signature comparison, and delivery-claim ledger stay as they are. The bridge webhook contract currently forbids non-public URLs and requires operator-supplied public addresses; this task provides that address from the gateway instead.

### Relevant Files

- `internal/automation/trigger_webhook.go` — `FormatWebhookEndpoint` (around L66) plus signature and timestamp validation, all reused unchanged
- `internal/automation/manager_triggers.go` — `DeleteTrigger` (around L178) is the insertion point for transactional binding cleanup
- `internal/daemon/bridges.go` — bridge subscription deletion, the bridge-side cleanup insertion point
- `internal/store/globaldb/global_db_bridge_bindings_routes.go` — existing binding-route storage to model the gateway binding repository on
- `internal/automation/trigger_lifecycle.go` — the webhook handling entry point and its ordered verification steps
- `internal/automation/trigger_delivery_claim.go` — the replay/delivery-identity ledger
- `internal/automation/manager_webhook_secrets.go` — webhook secret refs and resolution
- `internal/api/core/automation_trigger_handlers.go` — the delivery handlers and their scope split
- `internal/api/core/automation_request.go` — HTTP request construction, headers, and body bounds
- `internal/api/httpapi/routes.go` — webhook routes registered ahead of the guards; the public tier matrix must include them
- `internal/bridges/contract/webhook.go` — public URL validation rules the gateway address must satisfy
- `internal/bridges/control.go` — bridge webhook registration control method
- `internal/api/httpapi/bridge_routes.go` — bridge management routes and health surfaces
- `internal/workspaceaccess/` — the canonical policy used to authorize agent callers across workspace boundaries
- `internal/agentidentity/` — caller identity resolution feeding the authorization decision
- `internal/store/globaldb/schema/definitions/`, `internal/store/globaldb/schema/migrations/` — the appended fragment and migration for the binding table
- `internal/events/registry_base.go`, `internal/events/components.go` — event registration for binding lifecycle

### Dependent Files

- `internal/gateway/` — binding storage, generation invalidation, and the reconcile-time sweep live here
- `openapi/compozy.json`, `web/src/generated/compozy-openapi.d.ts` — regenerated for the binding endpoints
- `packages/site/content/docs/automation/webhooks.mdx` — delivery URL guidance plus the durability boundary
- `packages/site/content/docs/bridges/setup.mdx` — bridge binding replaces manual proxy instructions

### Related ADRs

- [ADR-004: Public tier surface set](adrs/adr-004.md) — ingress is the default public surface
- [ADR-010: Gateway state ownership](adrs/adr-010.md) — binding table shape, workspace exception, endpoint generation
- [ADR-001: First-party managed connectivity](adrs/adr-001.md) — why bridges stop needing operator-built proxies

## Deliverables

- Ingress binding table with scope ownership and endpoint generation, plus appended migration and generated sqlc output
- Per-trigger public delivery URL projection with honest off-state reporting
- Workspace-authorized binding create/delete with cross-workspace denial and transactional cleanup
- Bridge instances bindable to the gateway public address, with ingress reachability in bridge health
- Ingress management routes with contract types, regenerated artifacts, and registered events
- Documentation of the delivery URL and the uptime-bounded durability limitation
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-090, UT-091 — delivery URL projection for new and pre-existing triggers; honest off-state and empty-trigger surfaces
- [x] UT-092, UT-093 — per-instance bridge binding with confirmation; public-off health and external-proxy coexistence
- [x] UT-094 — endpoint-generation bump invalidating confirmations and flagging dependents
- [x] UT-095, UT-096, UT-097 — server-side owner resolution, cross-workspace denial, transactional cleanup with orphan sweeping
- [x] IT-040, IT-041, IT-042, IT-043 — signed delivery dispatch, distinct rejection classes, rate-bound behavior, duplicate-delivery race
- [x] IT-044, IT-045 — trigger and delivery attribution on the resulting run; offline-daemon sender-visible failure
- [x] IT-046 — ingress verification independent of operator authentication, including browser-visit rejection
- [x] IT-047, IT-048 — platform callback reaching bridge verification; address change flagging trigger URLs and bound bridges
- [x] IT-049, IT-050, IT-051 — cross-workspace agent denial, delete/recreate leaving no orphan, address stability across restart
- [x] E2E-003 — public ingress enabled, signed delivery to the shown URL, Loop run started with trigger and delivery attribution

## Success Criteria

- Every assigned test case implemented and passing
- A signed delivery to the displayed URL starts a Loop run attributed to its trigger and delivery
- A test proves an agent from another workspace cannot bind or unbind a foreign subject
- Deleting a subject leaves no orphaned binding, verified by a delete/recreate cycle
- Webhook verification behavior is byte-for-byte unchanged from before this task
- `make codegen-check` reports no drift after regeneration
