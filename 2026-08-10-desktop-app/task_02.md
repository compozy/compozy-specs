---
status: completed
title: "Go surfaces: compozy app verbs, config lifecycle, detection, quiesce contract"
type: backend
complexity: high
---

# Task 2: Go surfaces: `compozy app` verbs, config lifecycle, detection, quiesce contract

## Overview

Delivers every daemon/CLI-side surface the desktop capability needs: the `compozy app` verb tree with schema-validated structured output, the `[app]` config section with its full lifecycle, `desktop-app` install-method detection with the recommendation mapper, and the one bounded daemon contract change — the quiesce/safe-to-stop readout — closed end-to-end through codegen. After this task, agents can inspect the desktop surface and the runtime can truthfully report app-provisioned ownership.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST implement `compozy app open [path] | status | update [--check|--apply <app|runtime>] | retry | diagnose` per TechSpec Core Interfaces: `AppStatusReport` (schema-versioned, validated against `desktop/schema/app-state.schema.json`), platform-registration-derived `installed`/`app_version` (never from `app.json`), deterministic error identities (`app_not_installed`, `app_not_running`, `app_launch_failed`, `invalid_target_path`, `app_state_schema_unknown`, `app_control_unavailable`).
2. MUST register `newAppCommand` in `internal/cli/root.go` and route output through `writeCommandOutput` (`-o human|json|jsonl|toon`).
3. MUST implement the control-socket client (`app_control.go`) for `update|retry|diagnose` against `$COMPOZY_HOME/app.sock` (server lands in task_03 — client errors deterministically when absent/unresponsive).
4. MUST add `AppConfig` (`[app] update_check`, `update_check_interval` with bounds `[15m,168h]`) following the per-section pattern: own struct file, `defaultAppConfig()` wired into `DefaultWithHome`, `Validate()` with `ValidationError{Path,...}`, `config.toml` example, and the `[app]` section in the hand-authored `packages/site/content/docs/configuration/config-toml.mdx` — same change.
5. MUST add `InstallMethodDesktopApp = "desktop-app"` (`types.go`), the provenance-marker branch in `detect.go` (path+hash agreement; mismatch/unreadable ⇒ fall through), and the app-naming recommendation in the `manager_state.go` mapper — `managed: true` end-to-end through `GET /api/settings/update`.
6. MUST close the quiesce readout contract change: drain state + safe-to-stop (authoritative claims settled) machine-readable via the status/drain surface — `internal/api/contract` → `internal/api/core` → HTTP + UDS registration → `make codegen` (OpenAPI + generated TS) → generated docs, in this one task. No partial surface.
7. MUST regenerate CLI docs (`make codegen` emits `packages/site/content/docs/cli/app/`) and add the `cli/meta.json` navigation entry.
8. NO schema/SQLite change; NO new daemon route beyond the quiesce readout.
</requirements>

## Subtasks

- [x] 2.1 `internal/cli/app.go` + `app_types.go`: verb tree, status/open, schema validation, error identities
- [x] 2.2 `internal/cli/app_control.go`: control-socket client + deterministic absence errors
- [x] 2.3 Platform-registration resolvers for `installed`/`app_version` (macOS bundle query, Windows uninstall key, Linux `.desktop`)
- [x] 2.4 `AppConfig` lifecycle: struct file, defaults, validation, example, config-toml.mdx section
- [x] 2.5 `InstallMethodDesktopApp` + detect branch + recommendation mapper entry
- [x] 2.6 Quiesce readout contract: contract types, core handler, HTTP/UDS registration, `make codegen`, generated docs
- [x] 2.7 Cross-language gates: Go-side validation of `desktop/schema/` fixtures + shared config validation corpus
- [x] 2.8 CLI docs regen + `cli/meta.json` entry

## Implementation Details

Registration pattern: `internal/cli/root.go:170-185` (`newAppCommand(deps)` beside `newOpenCommand`). Config pattern: `internal/config/gateway_config.go` (struct+defaults+validate in one section file), root struct at `internal/config/config_extensions_sandbox.go:75-108`, defaults at `internal/config/defaults.go`. Detection: `internal/update/detect.go:67-97` + `internal/update/types.go:68-81` + recommendation strings in `internal/update/manager_state.go`. Drain surfaces: `internal/daemon/drain.go`, `internal/api/core/drain.go`, routes at `internal/api/httpapi/routes.go:38-40` (privileged) — extend the readout there. Handler home discipline: `internal/api/core` `BaseHandlers`; HTTP/UDS only register.

### Relevant Files

- `internal/cli/root.go` — verb registration
- `internal/cli/open.go`, `internal/cli/status.go`, `internal/cli/update.go` — output/error conventions to mirror
- `internal/config/config_extensions_sandbox.go`, `internal/config/defaults.go`, `internal/config/gateway_config.go` — config wiring points + pattern
- `internal/update/types.go`, `internal/update/detect.go`, `internal/update/manager_state.go` — detection + recommendation
- `internal/api/contract/status.go`, `internal/api/core/drain.go`, `internal/api/httpapi/routes.go`, `internal/api/udsapi/routes.go` — quiesce readout closure
- `desktop/schema/` — canonical schema + fixtures (from task_01)
- `packages/site/content/docs/configuration/config-toml.mdx`, `packages/site/content/docs/cli/meta.json` — docs surfaces

### Dependent Files

- `openapi/compozy.json`, `web/src/generated/compozy-openapi.d.ts` — regenerate via `make codegen` (contract change)
- `config.toml` (shipped example) — gains `[app]`
- task_03's Rust quiesce client — consumes the readout shape defined here

### Related ADRs

- [ADR-006](adrs/adr-006.md) — `compozy app` verb decision
- [ADR-010](adrs/adr-010.md) — config keys + fail-closed policy (amended)
- [ADR-011](adrs/adr-011.md) — decisions 2 (detection) and 7 (quiesce contract)

## Deliverables

- `compozy app` fully operable (open/status now; update/retry/diagnose clients erroring deterministically until task_03's socket exists)
- `[app]` lifecycle complete (struct→docs), `desktop-app` detection + recommendation end-to-end, quiesce readout closed with codegen
- Generated CLI docs + nav entry; `make codegen-check` clean
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [x] UT-071–UT-076 — status/open verb contracts, golden reports, transitional states
- [x] UT-077, UT-078 — AppConfig defaults + validation bounds
- [x] UT-079–UT-081 — desktop-app detection branch + fall-throughs
- [x] UT-082 — dead-PID app.json → running:false
- [x] UT-107 — platform-registration installation truth
- [x] UT-111 — control-socket client deterministic errors
- [x] UT-112 — recommendation mapper renders desktop-app end-to-end
- [x] IT-021 — Go reads Rust-written app.json fixtures (schema gate)
- [x] IT-022 — shared config validation corpus parity
- [x] IT-023 — detection through the settings handler (`GET /api/settings/update`)
- [x] IT-032 — quiesce readout HTTP/UDS parity

## Web/Docs Impact

`packages/site`: generated `cli/app/` docs + `cli/meta.json` entry + `[app]` section in `configuration/config-toml.mdx` (this task); `web/src/generated/compozy-openapi.d.ts` regenerates (no hand-written web change). **QA impact:** flag only — new CLI verbs are user-visible; scenarios planned in task_06, walked in task_07.

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: no extension/hook/registry contract change; generated OpenAPI/TS ripple only. Checked: manifests, provide surfaces, Host API, bridge SDKs, MCP.
- Agent manageability: THIS task is the agent surface — verbs, schema-versioned output, deterministic errors, HTTP/UDS parity for runtime truth.
- Config lifecycle: complete `[app]` lifecycle in this task (struct, defaults, validation, example, docs, tests).

## References

- TechSpec §Core Interfaces (Go blocks), §API Endpoints, §Config Lifecycle
- `analysis/04_analysis_compozy-runtime.md` + runtime-verify HEAD corrections (routes/middleware/verbs inventory)
- `analysis/07_analysis_tauri-dist-release.md` §6 (channel model context for status fields)

## Success Criteria

- Every assigned test case implemented and passing
- `compozy app status -o json` validates against the canonical schema in CI; `compozy config` shows `[app]` with docs parity
- `make codegen-check` and scoped `make gate` lanes green
