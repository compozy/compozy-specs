---
status: pending
title: Lifecycle and Selection Surfaces (CLI + API + Docs)
type: backend
complexity: high
---

# Task 4: Lifecycle and Selection Surfaces (CLI + API + Docs)

## Overview

Ships everything an operator or agent touches for lifecycle and selection: the root `--profile` flag + `COMPOZY_PROFILE`, the complete `compozy profile` verb group (list/current/use/create/update/rename/archive/unarchive/delete/ops/ops retry), the profiles/selection/plan routes on both transports with typed errors and remote-tier write rejection, resolution JSONL frames, lifecycle events — plus this capability's finished docs and official-skill content (vertical-slice rule). The IT-038 total-removal and IT-077 availability suites are born here and extended by tasks 08/09/10.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

**Skills**: `eng-code-guidelines`, `golang-master`, `eng-contract-codegen-coship`, `eng-test-conventions`, `testing-boss`, `documentation-writer`.

<requirements>
- MUST match `_dx.md` verbatim: verb output, exit codes, JSON payloads, error codes/messages/actions, plan flows (each mutating verb fetches its plan, shows it, submits `plan_revision` — also under `--yes` and non-TTY).
- MUST implement handlers as `BaseHandlers` methods in `internal/api/core` (shared HTTP/UDS registration, no transport-duplicated validation), with `internal/api/spec/` entries per route (`registry_<domain>.go` pattern) and regenerated TS types (`make codegen`).
- MUST carry `plan_revision` in the JSON body for POST rename/archive and as a query parameter for DELETE; stale → `409 profile_plan_stale`.
- MUST reject every profile-state write from the remote surface tier with `403 profile_remote_management_forbidden` (lifecycle mutations, `PUT /api/profiles/selection`, enablement writes when they exist) — reads stay scoped/labeled-aggregate only.
- MUST register every `_dx.md` error code as a typed payload in `marshalStructuredExecutionError`'s registry; CLI failures exit 1 and emit the same `{"error":{code,message,action}}` under `-o json`.
- MUST declare `--profile` once as a root persistent flag, join profile resolution to the workspace-resolution boundary (source taxonomy, context record, typed errors), and emit the profile-resolution JSONL frame (mirroring the workspace frame, including on empty lists).
- MUST keep machine commands (`daemon`, `doctor`, `update`) ignoring profile selection entirely (E2E-012).
- MUST extract new files instead of appending: `internal/cli/root.go` is at 499/500 lines — the verb group, error registrations, and flag wiring land in new files.
- MUST add the profile route set to the canonical transport-parity family (`internal/api/httpapi/transport_parity_integration_test.go` / `internal/api/udsapi/transport_parity_integration_test.go`) — extend, do not create a new harness.
- MUST emit the canonical events with correlation keys (`profile.created|renamed|identity_updated|archived|unarchived|deleted`, `profile.selection_changed`, `profile.lifecycle_op_recovered/failed`, `profile.plan_stale`) — never carrying secret refs.
- MUST ship the capability's docs (site profiles section: lifecycle, selection precedence, errors) and official-skill updates in the same task; CLI reference pages are generated (`make cli-docs` → `content/docs/cli/profile/`) — never hand-authored. Note the pre-existing `layout-profile` CLI group: `compozy profile` is a new adjacent group, registered in `registerRootCommands`.
- Rename's apply MUST execute the vault `profiles/<old>` ref-rewrite step against the Manager's explicit rewrite list (grammar registration is minimal here; the full credentials capability is task_08).
</requirements>

## Subtasks

- [ ] 4.1 Root `--profile`/`COMPOZY_PROFILE` wiring: persistent flag, resolver call after workspace resolution, context record, typed errors, machine-command exemption (new CLI files; root.go extraction).
- [ ] 4.2 `compozy profile` verb group: list (table + `-o json` with surface-decorated `current`), current (+ source/note), use, create (+ activate), update, rename (+ `--repos`), archive/unarchive, delete (+ enumeration confirm), ops + ops retry.
- [ ] 4.3 API: profiles CRUD + rename/archive/unarchive/delete + plan GETs + selection GET/PUT (single + full map) + ops routes; spec files; both listeners; remote-tier rejection.
- [ ] 4.4 Error registry + output bundle: every `profile_*` code typed; profile-resolution JSONL frame; empty-list frame parity.
- [ ] 4.5 Events with correlation keys; extend the observability coverage fixture (final completeness assertion IT-079 lands in task_09).
- [ ] 4.6 Bear the IT-038 suite: delete flow's preview == applied == result, field for field, over the rows existing at this phase (folders, memory rows + directory, selections, name release); leave explicit fixture-extension seams for tasks 08/09/10.
- [ ] 4.7 Bear the IT-077 suite: unavailability across selection/creation-trigger/ops surface/boot ordering; leave the discovery/prestart arms for task_08.
- [ ] 4.8 Extend canonical CLI suites (`workspace_test.go` boundary + a new sibling profile suite near `session_test.go` at 3296 lines) and the transport-parity family with the profile route set (IT-063).
- [ ] 4.9 Docs + skill: site profiles pages (lifecycle, selection, errors), `skills/compozy/references/` update (new `profiles.md` or `runtime-operations.md` + `configuration.md` entries), `make cli-docs` regen.
- [ ] 4.10 Flag QA scenarios for the new verbs/routes (content-addressed, `untested`).

## Implementation Details

`_dx.md` is the frozen contract — transcripts are E2E fixtures. The resolver joins the exact seams in `internal/cli/workspace_resolution.go`; the four-format output bundle contract lives in `internal/cli/format.go`.

### Relevant Files

- `internal/cli/workspace_resolution.go:15-24,84-146,338-393` — ladder, typed error payload, session-identity derivation, context record.
- `internal/cli/root.go:63-106,166-191,273-330` — deps injection (44 fns), the single persistent-flag site, `registerRootCommands` (57 constructors), `marshalStructuredExecutionError` (499 lines — extract).
- `internal/cli/format.go:63-175` — output bundle + JSONL frame + `--json`×`-o` conflict.
- `internal/api/core/` — `BaseHandlers` home for the new handler set.
- `internal/api/spec/` — `registry_workspaces.go`/`registry_sessions.go` as the registry pattern; add `registry_profiles.go` + schema/values files.
- `internal/api/httpapi/routes.go` + `internal/api/udsapi/routes.go` — hand-mirrored registration (HTTP has the local/remote surface-set split UDS lacks — the remote rejection rides it).
- `internal/api/httpapi/transport_parity_integration_test.go` + `internal/api/udsapi/transport_parity_integration_test.go` — the canonical parity family IT-063 extends.
- `internal/cli/workspace_test.go:331-410` + `internal/cli/session_test.go` — canonical CLI suites (session_test.go is 3296 lines; place profile cases in a sibling suite).
- `skills/compozy/SKILL.md` + `references/` (14 files) — official-skill update targets.
- `packages/site/content/docs/` — profiles docs section home; `content/docs/cli/` is generated output.

### Dependent Files

- `web/src/generated/compozy-openapi.d.ts` — regenerated; task_05 consumes.
- `internal/daemon/` — handler deps wiring + interface assertions against `*profile.Manager`.
- `internal/vault/types.go` — minimal `profiles` namespace registration for the rename rewrite (full capability in task_08).

### Competitor References

- `.resources/hermes/hermes_cli/profiles.py:1-58` — verb anatomy and lifecycle ergonomics; reject the `default` asymmetry.

### Related ADRs

- [ADR-001](adrs/adr-001.md) — archive/delete flows the verbs expose.
- [ADR-003](adrs/adr-003.md) / [ADR-014](adrs/adr-014.md) — selection precedence; daemon-owned remembered choice + routes.
- [ADR-010](adrs/adr-010.md) — local-only management; remote write rejection.
- [ADR-012](adrs/adr-012.md) — rename rewrites (vault refs) inside apply.

## Deliverables

- Complete `compozy profile` group + root selection flags, byte-faithful to `_dx.md`.
- Profiles/selection/plan/ops routes on both transports, spec files, regenerated types, parity coverage.
- Typed error registry + resolution frames + lifecycle events.
- Capability docs + official-skill content; generated CLI reference.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] UT-022, UT-076, UT-078, UT-079 — flag context record; `-o json` payloads; empty-list resolution frame; error-code mapping.
- [ ] IT-027, IT-028, IT-029, IT-030 — selection persistence, re-register reset, archive-keeps/delete-sweeps + notes, Global-lens slot.
- [ ] IT-031, IT-032, IT-033 — create visibility both listeners; create/delete races; concurrent identity edits.
- [ ] IT-034, IT-035 — rename protocol end-to-end (vault rewrites, selections untouched); repo offers/dormancy/wake.
- [ ] IT-036, IT-037, IT-038 — archive blocked/paused/frozen; unarchive restore; delete total enumeration (fixture extended by 08/09/10).
- [ ] IT-061, IT-062, IT-063, IT-064 — remote write rejection; selection/current inspection parity; profile route parity; API defaults/conflicts.
- [ ] IT-077 — availability gate (discovery/prestart arms extended by task_08).
- [ ] E2E-003, E2E-004, E2E-010, E2E-012 — lifecycle guardrails; two terminals; phase-0 journey; machine-command immunity.

### Web/Docs Impact

- `web/`: `web/src/generated/compozy-openapi.d.ts` (regen — task_05 consumes the new routes); no components in this task.
- `packages/site`: new profiles docs pages (lifecycle, selection precedence, errors) under `content/docs/`; `content/docs/cli/profile/` via `make cli-docs`; `content/docs/configuration/index.mdx` cross-link if selection env var is documented there.
- QA impact: new scenarios — add content-addressed untested files for profile create/switch/rename/archive/delete CLI journeys, selection precedence, ops/retry, and remote write rejection. Walk owned by task_13.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none here — manifest/enablement surfaces are task_09 (checked: extension manifest chain, registries, bridge SDKs untouched).
- Agent manageability: this task IS the plan — full verb group with structured output, HTTP/UDS parity, deterministic `profile_*` errors, ops discovery (`compozy profile ops`), resolution provenance (`profile current` source/note).
- Config lifecycle: no new keys; `COMPOZY_PROFILE` env documented with the capability docs; machine commands' selection immunity stated.
