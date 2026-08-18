---
status: completed
title: "Tauri shell: windows, native integration, runtime resolution"
type: desktop
complexity: high
---

# Task 1: Tauri shell: windows, native integration, runtime resolution

## Overview

Delivers the entire `desktop/` Rust crate up to "the app opens, resolves a runtime, and shows the product": the two-window shell (bundled `boot` + external-origin `main` with zero page IPC), every native affordance (single instance, deep links, window state, navigation fencing, external-link handoff), and the attach-first runtime resolution ladder (discovery, bound identity probe, install resolvers, detached supervisor). It also establishes the canonical cross-language state schema (`desktop/schema/`) that Task 2's Go surfaces consume.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST pin `tauri` 2.11.5 (hard security floor `>= 2.11.1`, CVE-2026-42184) and the exact plugin versions from analysis 06's Cargo pin; `Cargo.lock` committed; never `tauri-plugin-deep-link` 2.4.8.
2. MUST ship the single durable identity `com.compozy.os` / product name "CompozyOS" (ADR-006/012) — the identifier is load-bearing across five subsystems and never changes.
3. MUST implement the two-window shape (ADR-007 amended): `boot` (bundled pages, DESIGN.md token values, minimal capability) + `main` (daemon origin, created hidden, **no capability file matches it, no `remote` grant, `withGlobalTauri: false`**, zero registered commands beyond boot's needs).
4. MUST fence `main` navigation via `on_navigation` + `on_new_window` (allowed: daemon origin + bundled pages; everything else cancels and opens in the OS browser); `on_web_resource_request` MUST NOT be used (never runs for external URLs).
5. MUST implement the resolution ladder per TechSpec Core Interfaces: discovery (PID+start-time liveness), **bound** identity probe (real `StatusPayload` shape — date-string `schema_version`, nested `daemon.*` — agreeing with `daemon.json`; shaped-but-unbound ⇒ Foreign), install resolvers (app-owned → daemon.json-live → PATH via fix-path-env), supervisor (detached spawn `setsid`/`CREATE_NEW_PROCESS_GROUP`, stdio→log file, 30s readiness, 3-attempt backoff, quit never signals any daemon — BR-2).
6. MUST write `app.json` atomically with the canonical vocabulary (`resolving|provisioning|starting|attaching|product|updating|disconnected|skew|error`) validating against `desktop/schema/app-state.schema.json`; typed errors only (`ShellError{code, safe_message ≤512B, log_path}`) through the single redaction gate `errors.rs`.
7. MUST register single-instance FIRST (argv forwarding), deep links `compozyos://` (+`compozyos-dev://` for dev builds; `register_all()` every launch on Linux), window-state (denylist `boot`, `minWidth`/`minHeight` set, flash-free reveal, 10s load deadline → error state).
8. MUST read `[app]` config from `$COMPOZY_HOME/config.toml` with the fail-closed policy (absent ⇒ defaults; invalid ⇒ `ConfigInvalid`, update consumer off) and mirror `resolveHomeDir` exactly (env → `~/.compozy`; no workspace-`.env` overlay).
9. MUST add `make desktop-dev|desktop-build|desktop-test|desktop-lint` (cargo fmt+clippy `-D warnings`+test) and a CI test lane; `desktop/` stays outside the Bun/turbo graph and `mage Boundaries`.
10. Every file follows the TechSpec Architectural Boundaries split — one responsibility per file, hard cap 500 lines.
</requirements>

## Subtasks

- [x] 1.1 Crate scaffold: `tauri.conf.json` (identity, windows, capabilities), `Cargo.toml` pins, `build.rs` command allowlist, bundled `boot` pages
- [x] 1.2 Pure state machine (`state.rs`) + typed errors and redaction gate (`errors.rs`)
- [x] 1.3 Navigation fencing (`nav.rs`) + deep-link validation and last-wins queue (`links.rs`) + opener handoff
- [x] 1.4 Single-instance + deep-link + window-state + log plugins wired in `main.rs` (registration order, schemes, denylist, reveal)
- [x] 1.5 Home resolution (`home.rs`) + `[app]` config reader (`config.rs`) + `app.json` writer (`record.rs`)
- [x] 1.6 Canonical schema `desktop/schema/app-state.schema.json` + shared fixture corpus; Rust-side validation tests
- [x] 1.7 Discovery + bound probe + install resolvers (`runtime/discovery.rs`, `runtime/probe.rs`, `runtime/resolver.rs`)
- [x] 1.8 Supervisor: detached spawn, readiness, crash backoff, quit contract, log-tail escalation (`runtime/supervisor.rs`)
- [x] 1.9 Boot⇄main orchestration (`windowing.rs`): boot flow through resolving→attaching→product; disconnected/skew states
- [x] 1.10 Make targets + CI lane (fmt/clippy/test); stub-daemon + fake-daemon test fixtures under `desktop/tests/`

## Implementation Details

Follow TechSpec §Architectural Boundaries for the exact file split and §Core Interfaces for the Rust signatures (`ShellState`, `BoundDaemonIdentity`, `RuntimeResolver`). The two-window pattern, capability emptiness, navigation hooks, spawn mechanics, and window-state details are specified in `analysis/06_analysis_tauri-core-verified.md` §§1–3, 5–8, 10 — read it in full before writing Rust.

### Relevant Files

- `desktop/src-tauri/**` — new crate (all modules per TechSpec layout)
- `desktop/schema/app-state.schema.json` + `desktop/schema/fixtures/` — canonical cross-language contract (new)
- `internal/daemon/info.go`, `internal/cli/daemon_status.go:76-93` — the discovery/liveness algorithm to mirror (read-only)
- `internal/api/contract/status.go` — the real status payload shape the probe decodes (read-only)
- `internal/config/home.go:145-158` — `resolveHomeDir` rules to mirror (read-only)
- `internal/config/gateway_config.go` — per-section config file pattern the `[app]` reader's defaults must match (read-only)
- `Makefile` — new desktop targets
- `.github/workflows/` — desktop test lane

### Dependent Files

- `task_02`'s Go readers (`internal/cli/app_types.go`) — consume `desktop/schema/` fixtures
- `packages/ui/src/tokens.css` / `DESIGN.md` — token values for the bundled boot pages (values copied, not imported)

### Related ADRs

- [ADR-007](adrs/adr-007.md) — window strategy, two-window amendment, fencing, fixed port
- [ADR-002](adrs/adr-002.md) — attach-first ladder
- [ADR-009](adrs/adr-009.md) — deep-link path pass-through + schemes
- [ADR-010](adrs/adr-010.md) — `[app]` config + fail-closed policy
- [ADR-012](adrs/adr-012.md) — durable identity boundary

## Deliverables

- Buildable, signed-nothing-yet `desktop/` crate: opens boot, attaches to a running daemon or starts an installed one, shows the product at the daemon origin, honest error/skew/disconnected states
- Canonical schema + fixture corpus consumed by both languages
- Make targets + green CI lane (fmt, clippy `-D warnings`, tests)
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] UT-001–UT-009 — ShellState machine transitions + error mapping
- [x] UT-010–UT-012 — home resolution (match-Go semantics, shared fixture table)
- [x] UT-013–UT-016 — discovery + liveness + stale/malformed records
- [x] UT-017–UT-023, UT-089–UT-091 — bound identity probe + handshake + squatter/PID-reuse defenses
- [x] UT-024–UT-028 — supervisor spawn/readiness/quit/stop-guard
- [x] UT-039–UT-041 — `[app]` config defaults + fail-closed + bounds
- [x] UT-042, UT-043, UT-088, UT-099 — app.json atomic writes, transitional states, write-failure, typed errors
- [x] UT-044–UT-048 — deep-link validation + queue
- [x] UT-049–UT-051 — navigation fencing + external handoff
- [x] UT-052–UT-055 — window-state restore/recovery + load deadline
- [x] UT-087 — error-state escalation with log tail
- [x] UT-103, UT-104 — redaction gate adversarial corpus
- [x] UT-105, UT-106 — install resolver precedence + GUI-environment resolution
- [x] IT-001, IT-002, IT-004–IT-010 — resolution ladder, squatter, skew, crash/quit contracts (stub + fake daemon)
- [x] IT-019, IT-020 — deep link navigation + single-instance arg forwarding
- [x] IT-024, IT-026–IT-028 — start race, stale record, isolated homes, unhealthy-runtime retry

## Web/Docs Impact

None in this task — backend/shell only; site/docs land in task_04, CLI docs in task_02. **QA impact:** flag only — new user-visible behavior (install/attach/deep-link/single-instance) gets `untested` scenarios planned in task_06 and walked in task_07 (loop rule: intermediate tasks flag, the QA phase walks).

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — no extension/hook/registry surface touched (shell is a daemon client). Checked: extension host API, hooks, provide surfaces, bridge SDKs.
- Agent manageability: produces the `app.json` + schema that `compozy app status` (task_02) reads; no CLI/HTTP surface in this task.
- Config lifecycle: consumes `[app]` (Rust side); the Go struct/validation/docs land in task_02 — defaults must match the shared fixture corpus by test.

## References

- `analysis/06_analysis_tauri-core-verified.md` (primary), `analysis/04_analysis_compozy-runtime.md` + TechSpec HEAD corrections, `_brief.md` R1/R2/R4
- `.resources/foundry/packages/desktop/` · `.resources/openfang/crates/openfang-desktop/` (capabilities layout) · `.resources/librefang/crates/librefang-desktop/` — working Tauri v2 references (gitignored; cite, do not assume present in CI)

## Success Criteria

- Every assigned test case implemented and passing
- `make desktop-build` produces a runnable dev app on the host platform; attach/start/provision-needed paths reach their designed states with a real `compozy` daemon
- Zero capabilities granted to the `main` window (verified by test + config review); clippy/fmt clean
