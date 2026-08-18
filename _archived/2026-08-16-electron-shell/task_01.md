---
status: completed
title: Update Operation, single command, and daemon surfaces
type: backend
complexity: critical
---

# Task 1: Update Operation, single command, and daemon surfaces

## Overview

Delivers Part II phase P1 end to end on the Go side while the Tauri app keeps shipping untouched: the durable Update Operation with its detached coordinator (ADR-009), the unified self-update mechanism extended to desktop-app installs, the single `compozy update` command (deleting `compozy app update`, ADR-005), the extended settings update surface (GET/POST/cancel with the three-projection contract), the Go bootstrap/supervision entry point, the app-state schema v2 step, and the daemon-served CSP. Every other task builds on the primitives this one lands.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST implement the Update Operation record + companion lock exactly per `_spec.md` Data Models: lock-serialized create-if-absent with fsynced atomic publication, monotonic `revision`, nullable fenced holder (`executor_generation`, `pid_start_time`, `lease_expires_at`), `waiting-for-app` dormancy, single-winner resume, dormant-only cancel, the per-phase recovery/cancel matrix, and idempotent terminal archive to `update-history.jsonl` (Safety Invariants 1–12).
2. MUST route every journal mutation through the single Go transition API `Transition(operation_id, executor_generation, expected_revision, transition)`; no peer writer exists anywhere (L-005).
3. MUST implement the detached coordinator as the sole runtime-mutation executor (verify → swap → restart → health-check → finalize/restore) surviving daemon generations; the daemon endpoint only acquires, spawns detached, and serves the journal (Safety Invariant 4).
4. MUST extend `internal/update` so `desktop-app` installs self-apply (provenance rewritten inside the swap's lock scope) and `RequestAppApply` records the verified asset identity + attempt id, returning `accepted` — never a terminal verdict.
5. MUST implement the compatibility gate as specified: `compat.json` (`{runtime_version, min_app_version}`) verified via the checksums catalog, coordinator backstop refusal for runtime-only requests, and `/api/status` gaining `min_app_version` (publication-side gate belongs to task_04).
6. MUST rewrite `compozy update` to the multi-target record with aggregate precedence `failed > blocked > staged > updated > available > up-to-date`, add `--cancel`, and delete `compozy app update` with its tests and control methods (`update.check`/`update.apply`) — no alias, no shim.
7. MUST implement `GET /api/settings/update`, `POST /api/settings/update/apply`, and `POST /api/settings/update/cancel` as `BaseHandlers` methods with HTTP+UDS parity, typed vocabularies end to end, the full projection listed in Part II API Endpoints, and OpenAPI + generated TypeScript co-shipped (`make codegen`).
8. MUST implement the three explicitly mapped projections (CLI terminal record with frozen `daemon_restarted`/`message`; live settings projection; app-status projection) from one computation, including the exhaustive phase→UI mapping table, each golden-tested against its surface contract.
9. MUST step the app-state schema to v2 exactly as enumerated (value-set extensions + `operation_id`/`phase`/`percent`), updating `desktop/schema/app-state.schema.json`, fixtures, and CLI validation in the same change; mismatches fail with the existing `app_state_schema_unknown` error.
10. MUST implement the Go bootstrap/supervision entry point (`compozy daemon bootstrap -o jsonl`): resolution (attach → start → provision), install-from-bundle self-provision, readiness, bounded retry/backoff with give-up classification, and typed phase events — bundle-aware but consumable with a test bundle before the shell exists.
11. MUST make the daemon the sole consumer of `[app] update_check`/`update_check_interval` for both tracks; no second scheduler or clamp implementation exists.
12. MUST serve the SPA with the exact production CSP of Security Invariant 16 (including `frame-ancestors 'none'`), with dev relaxations impossible in packaged serving.
13. MUST emit the canonical `update.*` event taxonomy at the transition call site only (never tailing history), with the required correlation fields and the UT-080 coverage-matrix discipline.
14. MUST keep `compozy app status/open/retry/diagnose` byte-compatible (except the v2 schema step) and the deterministic error taxonomy frozen.
15. MUST update the surgical CLI docs for the changed verbs in this same task (no partial surface): `cli/update.mdx` rewrite, `cli/app/update.mdx` deletion + meta/compozy row removal — the broad docs truth pass stays in task_05.
</requirements>

## Subtasks

- [x] 1.1 Operation record + companion lock + transition API (revision CAS, generation fencing, recovery/cancel matrix, history archive) in `internal/update`
- [x] 1.2 Detached coordinator: verify/swap/restart/health/finalize/restore across daemon generations; boot + launch + bounded periodic recovery sweeps
- [x] 1.3 Desktop-app self-apply + provenance ownership + `RequestAppApply` (verified identity, attempt id, accepted-only)
- [x] 1.4 Compatibility gate: `compat.json` consumption in the coordinator backstop + `/api/status` `min_app_version` handshake
- [x] 1.5 `compozy update` rewrite (multi-target record, `--cancel`) + `compozy app update` deletion (command, tests, control methods, surgical docs)
- [x] 1.6 Settings surface: GET/POST/cancel handlers, typed enums, HTTP+UDS registration, OpenAPI registry, `make codegen` co-ship
- [x] 1.7 Three-projection contract + phase→UI mapping, golden-tested against `_dx.md` transcripts
- [x] 1.8 App-state schema v2: schema file, fixtures, CLI validation, both status projections
- [x] 1.9 `compozy daemon bootstrap -o jsonl`: ladder policy, install-from-bundle, typed progress, supervision tables
- [x] 1.10 Daemon-only `[app]` cadence consumption for both tracks
- [x] 1.11 Daemon-served CSP (exact policy) on the SPA responses
- [x] 1.12 Event taxonomy at transition call sites + coverage-matrix test
- [x] 1.13 Full assigned test suite green; `make gate` + `REL-beta-self-update` re-walk flagged

## Implementation Details

Follow `_spec.md` Part II: System Architecture (component table), Implementation Design (Core Interfaces, Data Models, API Endpoints, Release compatibility gate), Safety Invariants 1–18, and Monitoring. The coordinator runs as a re-exec of the `compozy` binary behind an internal entry point; the shell (task_03) will consume the transition API only via the installed binary.

Skills to activate: `eng-code-guidelines`, `golang-master`, `eng-test-conventions`, `testing-boss`, `eng-consolidate-test-suites`, `eng-cleanup-failure-paths` (rollback/finalizer paths), `eng-contract-codegen-coship` (settings routes + OpenAPI/TS), `systematic-debugging` (writer races).

### Relevant Files

- `internal/update/{manager.go,manager_state.go,apply.go,detect.go,types.go,verify.go,github.go,release_track.go,cache.go}` — the existing mechanism this task extends (sigstore/TUF verify, backup/restore, install-method detection)
- `internal/cli/update.go`, `internal/cli/update_command_test.go` — single-command rewrite target (updateRecord → multi-target)
- `internal/cli/app.go`, `internal/cli/app_control.go`, `internal/cli/app_errors.go`, `internal/cli/app_types.go`, `internal/cli/app_test.go` — verb deletion, schema v2 validation, preserved-verb regression
- `desktop/schema/app-state.schema.json`, `desktop/schema/fixtures/*.json` — v2 step
- `internal/api/core/{interfaces.go,settings.go,base_handlers.go}`, `internal/api/{httpapi,udsapi}/routes.go`, `internal/api/udsapi/{server.go,handler_config.go,server_handlers.go}`, `internal/api/spec/registry_settings.go` — settings surface extension (existing `GetSettingsUpdate` is the wiring precedent)
- `internal/daemon/{boot_settings.go,settings.go,update_lock.go,restart_request.go,restart_relaunch.go,info.go,boot_components.go}` — composition root wiring, restart machinery the coordinator reuses, flock
- `internal/cli/daemon.go` — daemon start path the bootstrap verb extends
- `internal/config/{home.go,app.go,defaults.go,merge_app.go}` — home paths (formalize `bin/`, `app.json`, operation files) and `[app]` keys
- `internal/api/httpapi/static.go` — CSP header injection point
- `internal/version/version.go` — version truth for monotonicity checks

### Dependent Files

- `web/src/generated/compozy-openapi.d.ts` — regenerates; task_02 consumes
- `internal/cli/root.go` — command registration (app update removal)
- `packages/site/content/docs/cli/update.mdx`, `packages/site/content/docs/cli/app/{update.mdx,meta.json}`, `packages/site/content/docs/cli/compozy.mdx` — surgical verb-doc updates in this task
- `skills/compozy/` — update-command delta (official skill truth; completed fully in task_05)

### `.resources/` References

- `.resources/synara/apps/desktop/src/updateInstallMarker.ts` — durable install-attempt marker + failure-count escalation pattern the operation record's app section generalizes
- `.resources/synara/apps/desktop/src/backendSupervisionPolicy.ts` — bounded backoff + give-up-with-dialog supervision table shape (the `compozy daemon bootstrap` policy analog)
- `.resources/synara/apps/desktop/src/resumableUpdateDownload.ts` — known electron-updater/mac download-stall context informing the risk register (adopt only if observed)

### Related ADRs

- [ADR-004](adrs/adr-004.md) — one mechanism, two consumers
- [ADR-005](adrs/adr-005.md) — single command; deleted verb
- [ADR-009](adrs/adr-009.md) — operation record, coordinator, lease/fencing/resume/cancel
- [ADR-002](adrs/adr-002.md) — relocation + bundled-bootstrap note (this task builds the Go side)
- [ADR-007](adrs/adr-007.md) — lockstep release lookup the checks rely on

### Web/Docs Impact

- `web/`: `web/src/generated/compozy-openapi.d.ts` regenerates (extended GET payload + apply/cancel routes); no component/hook changes in this task (task_02 owns them; `settingsUpdateOptions`/`useSettingsUpdate` keep working against the additive GET).
- `packages/site`: `content/docs/cli/update.mdx` (multi-target + `--cancel`), `content/docs/cli/app/update.mdx` **deleted** + `content/docs/cli/app/meta.json` and `content/docs/cli/compozy.mdx` rows, generated API reference regen. Broad desktop-page pass stays in task_05.
- QA impact: reset to `untested` — `docs/qa/scenarios/REL-beta-self-update.md` (mechanism replaced) and `APP-agent-cli-app-verbs` (verb surface changed); add content-addressed `untested` scenarios for the new behaviors: single-command multi-target update, `--cancel`, settings apply/cancel routes (ids per `qa-execution` content-addressing). Flag then verify per the loop contract (walk happens in task_07).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked extension manifests, hooks, skills/capabilities, tools/resources, registries, bridge SDKs, MCP sidecars; no surface references updates or the shell; no extension point added (spec Part I constraint). `skills/compozy/` update-command delta ships here.
- Agent manageability: `compozy update [-o json|jsonl]` + `--check` + `--cancel`; `GET/POST /api/settings/update{,/apply,/cancel}` HTTP+UDS parity; deterministic statuses (`blocked` with holder, `staged`, `failed`) and frozen `compozy app` error taxonomy; cross-surface state comparison tests included (IT-011/IT-012/IT-024).
- Config lifecycle: `[app] update_check`/`update_check_interval` unchanged keys/defaults/validation; consumer set reduced to daemon-only (UT-072, IT-016); no new keys; `configuration/config-toml.mdx` untouched (verified — consumer prose lives in task_05's truth pass).

## Deliverables

- The Update Operation + coordinator running end to end on a real home (CLI foreground and daemon detached paths), with recovery proven per journaled phase
- `compozy update` matching every `_dx.md` transcript byte-for-byte; `compozy app update` gone
- Settings update surface live on HTTP+UDS with regenerated OpenAPI/TS (`make codegen-check` clean)
- App-state schema v2 shipped lockstep (schema, fixtures, validation, projections)
- `compozy daemon bootstrap` consumable with typed progress against a test bundle
- Daemon CSP serving per invariant 16
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-001–UT-018 — unified mechanism (check/apply/verify/rollback/provenance/lockstep)
- [x] UT-019–UT-026 — `compozy update` CLI record golden transcripts
- [x] UT-027–UT-032 — preserved `compozy app` verbs + schema v2 validation (status/open errors)
- [x] UT-035–UT-039 — settings controller (accepted/blocked/detached-spawn semantics)
- [x] UT-053–UT-054, UT-056–UT-060 — Go supervision + install-from-bundle policy tables
- [x] UT-072 — `[app]` config governance (daemon-only consumer)
- [x] UT-074–UT-077 — compat backstop, acquisition contention, transition/recovery matrix, digest binding
- [x] UT-079–UT-080 — projection completeness + event coverage matrix
- [x] UT-082–UT-083 — workspace-isolation evidence + lease/dormancy semantics
- [x] IT-001–IT-016 — daemon apply/restart/rollback, intent/record flows, transport parity, provenance, config
- [x] IT-018–IT-020 — cross-process writer races, journal recovery, compat backstop
- [x] IT-022 — daemon CSP protection (negative cases)
- [x] IT-024 — cancel parity across surfaces
- [x] E2E-023, E2E-026–E2E-028 — CLI journeys (transcripts, offline, headless, unknown-command)

## Success Criteria

- Every assigned test case implemented and passing
- `make gate` green including `-race` on the new concurrency paths; `make codegen-check` clean
- A kill-at-every-phase harness run leaves the home on a working runtime with a truthful journal (IT-019 evidence)
- `_dx.md` transcripts reproduced byte-identical by the golden tests
- No `compozy app update` reference survives outside deletion-context (grep evidence)
