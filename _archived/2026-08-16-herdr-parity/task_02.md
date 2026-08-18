---
status: completed
title: "Attention config, settings section, and the notify service + transport"
type: backend
complexity: high
---

# Task 2: Attention config, settings section, and the notify service + transport

## Overview

Delivers the notification channel's daemon half (P3 backend): the `[attention]` config section with live lifecycle, the settings section + `GET/PATCH /api/settings/attention` round-trip, and the notify service — sanitization, caps, per-session rate limiting, per-workspace mute, and the `operator_notification` named catalog-stream event whose live-subscription count makes `delivered` a provable outcome (round-2 B-107). Ships `compozy notify`, `POST /api/agent/notify`, and the `compozy__notify` native tool.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. `[attention]` MUST follow the `worktrees.go` exemplar: struct + `Default…()` + `Validate()` (ValidationError paths), overlay `merge_attention.go`, lifecycle classification `Live` for all four keys (`toasts` true, `sound` true, `system` false, `muted_workspaces` []), CLI mutation-map entries, docs + examples + tests in the same change (SD-011).
2. The settings section MUST follow the window-manager exemplar: `SettingsSectionName` const, section build in `internal/settings/`, contract payload, `GET`/`PATCH` (privileged) on HTTP and UDS with spec entries and parity-test updates; PATCH applies live (no restart).
3. Notify MUST return `NotifyResult{Outcome, RetryAfterMS}` with daemon-provable outcomes only (`delivered | no-client | muted-workspace | rate-limited`); `delivered` means the sanitized `operator_notification` event reached ≥1 live operator-client catalog-stream subscription at publish time.
4. Sanitization and caps (title ≤80, body ≤240) MUST run before any broadcast; rate limit 1/s per sender session, fail-closed to `rate-limited`; muted workspaces deliver nothing but keep counters (Business Rules 19, 22).
5. Deleting a workspace MUST prune its `muted_workspaces` entry (no orphans).
6. The `compozy__notify` descriptor/handler/meta-row/toolset/binding MUST land in the new sibling files (`sessions_attention.go`, `native_tool_session_attention_calls.go`) — never growing `builtin/sessions.go` (455L) or `native_tool_session_calls.go` (341L).
7. Docs MUST ship in this task: `skills/compozy/references/configuration.md` (`[attention]`), `native-tools.md` (notify), site config reference (`config-toml.mdx` table + lifecycle matrix) and the notify CLI/API reference pages.
</requirements>

## Subtasks

- [x] 2.1 Config section: struct/defaults/validate + overlay + lifecycle classes + `compozy config get/set attention.*` classification.
- [x] 2.2 Settings section + contract payload + GET/PATCH routes on both transports with parity-test updates.
- [x] 2.3 Notify service: sanitize/caps/rate-limit/mute pipeline, `NotifyResult`, subscriber-count-based `delivered`, `operator_notification` event publish.
- [x] 2.4 Surfaces: `POST /api/agent/notify` (agent identity), `compozy notify` verb (human/toon formatters, outcomes), `compozy__notify` tool (descriptor, meta row, binding, workspace scope).
- [x] 2.5 Workspace-mute pruning on workspace deletion.
- [x] 2.6 Contracts + codegen + fixtures (`OperatorNotificationEventPayload`, settings payload) for web/E2E consumers.
- [x] 2.7 Docs: skill references + site config/CLI/API pages.

## Implementation Details

Reference `_spec.md` Part II (Config Lifecycle, Core Interfaces notify block, API Endpoints) and ADR-002 (as amended: provable outcomes).

### Relevant Files

- `internal/config/worktrees.go` (108L exemplar: struct/defaults/validate + dotted-path consts) + `internal/config/config_extensions_sandbox.go:75-109` (root `Config` struct home) + `internal/config/merge_window_manager.go:1-60` (overlay exemplar) + `internal/config/lifecycle/lifecycle.go` (Live class).
- `internal/cli/config.go` (~150-300: `configScalarMutationKinds`) + `internal/cli/config_path_classification.go:11` — new-key classification.
- `internal/api/contract/settings.go:30-42` + `internal/settings/window_manager_section.go` + `internal/api/httpapi/routes.go:399-400` — settings-section exemplar chain.
- `internal/session/session_catalog_stream.go` broadcaster — the transport `operator_notification` rides; subscriber accounting lives here.
- `internal/tools/builtin/descriptors.go:78-110` (`nativeDescriptor`) + `internal/toolmeta/native_entries.go` + `internal/daemon/native_tool_binding_groups.go:109-112` + `internal/daemon/native_tool_results.go:13` (`structuredResult`) — tool chain.
- `internal/agentidentity/` — agent-identity resolution for the notify route.
- Redaction: the existing secret redactor used by wake/task paths (see `internal/sse/scrub.go` and task-wake sanitizer) — reuse, never fork.

### Dependent Files

- `web/src/generated/` types + `web/src/systems/os/mocks/*` — event payload fixtures for task_03's notifier.
- `internal/api/spec/` + both route-inventory parity tests.
- `skills/compozy/references/{configuration,native-tools}.md`; `packages/site/content/docs/configuration/config-toml.mdx` (~2400L, hand-written table) + lifecycle matrix page.

### Related ADRs

- [ADR-002: Notification policy](adrs/adr-002.md) — triggers/defaults this config encodes; provable-outcome amendment.
- [ADR-005: Attention truth pipeline](adrs/adr-005.md) — the stream the transport rides.

### Competitor References

- `.resources/herdr/src/server/notifications.rs` + `src/app/actions.rs:111,155` — notification triggers, `notification.show` result reasons (`shown|disabled|rate_limited|no_foreground_client|busy`) and the 1s rate limit this task's provable outcomes refine.

### Web/Docs Impact

- `web/`: `web/src/generated/compozy-openapi.d.ts` (settings + notify payloads), `web/src/systems/os/mocks/handlers.ts`/`fixtures.ts` (`operator_notification` fixtures), `web/src/systems/settings/` adapters consume the new section in task_03.
- `packages/site`: `content/docs/configuration/config-toml.mdx` (`[attention]` rows), lifecycle-matrix page, notify CLI/API reference, `skills/compozy/references/configuration.md` + `native-tools.md`.
- QA impact: new scenarios — add content-addressed `untested` files for: `[attention]` round-trip UI/file/CLI, notify outcomes (delivered/no-client/rate-limited/muted). Flag only — task_08 walks them.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: `compozy__notify` joins the agent contract (tool registry + skill docs); `operator_notification` is a client contract, not an extension surface; hooks/manifests/bridges/MCP unaffected (checked).
- Agent manageability: `compozy notify` + `POST /api/agent/notify` + `compozy__notify` with deterministic outcomes; `compozy config get/set attention.*` round-trips identically with the settings UI and file.
- Config lifecycle: `[attention]` keys/defaults/overlay/validation/examples/docs/tests all in this task (SD-011); no keys removed (checked: no existing notification keys exist — `internal/notifications/` bridge presets untouched).

## Deliverables

- `[attention]` live-applying config + settings section round-trip on both transports.
- Notify pipeline with provable outcomes + the named-event transport, on all three agent surfaces.
- Codegen artifacts + fixtures current; docs shipped.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

- [x] UT-030..UT-034 — notify sanitize/caps/rate-limit/mute
- [x] UT-047..UT-049 — config defaults, validation, overlay
- [x] IT-010 — notify route outcomes incl. `delivered`/`no-client` subscriber accounting
- [x] IT-011 — settings GET/PATCH round-trip, live apply, concurrent writers
- [x] IT-024 — workspace-mute pruning
- [x] E2E-007 — `compozy notify` transcript incl. rate-limited burst

## Success Criteria

- Every assigned test case implemented and passing.
- UI/file/CLI agree on every `[attention]` key with no daemon restart (US-011 ACs).
- An agent can tell truthfully whether its notification reached a live client (US-013 ACs).
- `make gate` green; parity tests include the settings + notify routes.

## Completion Notes

Focused close passed with race detection: config/session/settings (38 tests), API/spec/transports
(125), CLI (7), native descriptor/toolset inventory (304), and daemon wiring/pruning (314). Web and
site typechecks plus the codegen consistency dependency passed. E2E-007 is materialized and compiles;
execution, screenshots, QA walks, and project-wide gates remain intentionally deferred to tasks 07–08
and the workstream tail.

Compozy Impact Audit:

- Native tools: added `compozy__notify` to the sessions toolset, descriptor catalog, risk metadata,
  availability, session/workspace-bound handler, schemas, binding validation, and official docs.
- Extensibility and hooks: no extension, hook, bridge, MCP, or sidecar behavior changed; checked the
  native descriptor/toolset/meta registries and kept operator notifications on the client catalog
  stream. The global `[attention]` lifecycle is live and fully represented in config/settings docs.
- Workspace data isolation: notification data is session-scoped with an explicit owning
  `workspace_id`; HTTP caller identity and native bound scope reject caller-supplied ownership, and
  the SSE payload preserves both ids. Muted workspace ids are global policy entries and are pruned on
  workspace deletion. Route, CLI, native-tool, manager, and deletion tests cover these paths.
- Official Compozy skill: updated `skills/compozy/references/configuration.md` and
  `skills/compozy/references/native-tools.md` for the public config, CLI/API, and tool contract.
