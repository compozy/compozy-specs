---
status: pending
title: Expose surfaces and public projections closure
type: backend
complexity: high
---

# Task 4: Expose surfaces and public projections closure

## Overview

Makes exposure and origin public everywhere at once: the expose/unexpose endpoints with the single `expose_failed` envelope, the CLI verb family (`expose`, `unexpose`, `create --expose`, `info`, `where`, `list` origin column) with `_dx.md`-exact renderings, and the projections closure matrix — `origin` on the shared skill summary DTO across the list route, native tools, and the extension Host API, plus `exposures[]` on detail payloads. Closes the backend contract with transport parity and the observability coverage-matrix suite.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — no compat shims, no fallbacks, no placeholders (greenfield hard cuts)
</critical>

<requirements>
1. MUST implement `POST /api/skills/{name}/expose` and `/unexpose` in `internal/api/core` `BaseHandlers`, registered on BOTH transports: body `{targets[], workspace_id?}` (canonical id only; CLI resolves ID/name/path first); success 200 `{name, workspace_id?, results[], rolled_back}`; ANY failure → 409 with the ONE envelope (top-level `expose_failed` + per-target results; `rolled_back: true` when compensation completed; per-target cleanup errors otherwise); `workspace_id` echoed on workspace-scoped ops and omitted for global skills — identically placed in success and failure shapes. Unexpose returns per-target independent results with NO `rolled_back` field.
2. MUST enforce the authorization split: expose/unexpose pass the same gate as skill enable/disable (agent-scope callers allowed); source config keys stay rejected at agent scope (contrast covered by UT-093 vs UT-015).
3. MUST extend `GET /api/skills/{name}` (and `skill info`) with `origin` + `exposures[]{target, path, status}` using the four-state vocabulary; `GET /skills` list items gain `origin` (empty for compozy tier).
4. MUST close the public-projection matrix in this one change (spec closure table): shared skill summary DTO `origin` consumed by `listSkills`, `compozy__skill_list`, `compozy__skill_search`, and extension Host API `handleSkillsList` (+ `extension/contract/skills.go` type); `compozy__skill_view` gains `origin` + `exposures[]`; native-tool descriptors and schema digests refreshed; `SkillRuntimeStatusPayload` explicitly unchanged (existing status tests keep proving it).
5. MUST ship the CLI verb family exactly per `_dx.md` transcripts: `skill expose|unexpose` (success, idempotent "no change", failure renderings for target-disabled / name-conflict / multi-target-rolled-back — exit 1, `-o json` mirrors the API envelope); `skill create --expose` (partial success: created line + expose error + exit 1, skill exists on disk); `skill info` EXPOSED TO block with all four status renderings; `skill where` WINNER/ALSO/LINKS with qualified-form hints and `— none —` empty case; `skill list` ORIGIN column.
6. MUST keep HTTP/UDS parity by construction (shared `BaseHandlers`, no transport-duplicated parsing) and prove byte-equivalent field sets and error codes via the canonical parity suite (IT-005 covers settings scope shapes + expose endpoints).
7. MUST land the observability coverage-matrix suite (IT-014): walks every lifecycle path from tasks 02-04 (applied/superseded/apply_failed/truncated/link_skipped/exposure.*) and fails when any canonical event is missing, lacks base correlation keys, or when durable append does not precede the live broadcast for the same change.
8. MUST co-ship contract changes end-to-end: OpenAPI op specs, generated TS types, acpmock/matcher updates where session fixtures see new fields — `make codegen` + `codegen-check` green (runtime-contract co-ship rule, L-007).
9. SHOULD route all new handler logic through existing conversion/error-mapping seams (`conversions_skills.go`, `mapSkillScopeError`) rather than new parallel helpers.
</requirements>

## Subtasks

- [ ] 4.1 Expose/unexpose op specs + `BaseHandlers` methods + route registration on both transports
- [ ] 4.2 `expose_failed` envelope + `workspace_id` placement rules + unexpose independent-results shape
- [ ] 4.3 Authorization: expose/unexpose under the enable/disable gate; agent-scope contrast with config keys
- [ ] 4.4 Skill detail/list payloads: `origin` + `exposures[]`; shared summary DTO adoption across list route
- [ ] 4.5 Native tools closure: `skill_list`/`skill_search`/`skill_view` fields + descriptor/schema digest refresh
- [ ] 4.6 Extension Host API closure: `handleSkillsList` + contract type round-trip
- [ ] 4.7 CLI: `expose`/`unexpose` + failure renderings; `create --expose` partial-success semantics
- [ ] 4.8 CLI: `info` EXPOSED TO (four states), `where` (winner/also/links), `list` ORIGIN column
- [ ] 4.9 Observability coverage-matrix suite (IT-014)
- [ ] 4.10 Codegen + parity + CLI E2E journeys (E2E-001, E2E-005)

## Implementation Details

Follow `_spec.md` Part II — API Endpoints (shapes in `_dx.md` are the contract), the public skill-projection closure matrix, and Agent Manageability Plan. The manager from task_03 is the only mutation path; handlers translate `TargetResult`s into the wire envelope without re-deriving state.

### Relevant Files

- `internal/api/spec/registry_skills.go:5-212` — op-spec registry (9 existing skill ops; new ops follow this shape) + `skill_schema_enums.go`
- `internal/api/core/skills.go:18-450` — handler home (`resolveSkillScope:374`, `mapSkillScopeError:402`, post-mutation refresh `:427-450`)
- `internal/api/core/conversions_skills.go:16-120` — `SkillPayloadFromSkill` (origin/exposures projection point)
- `internal/api/contract/skill_payloads.go:16-110` — DTO field site (`Source` at `:28`; `origin` is additive)
- `internal/api/httpapi/routes.go:258-267,396-397` + `internal/api/udsapi/routes.go:285-295,409-410` — dual registration
- `internal/api/udsapi/transport_parity_integration_test.go` — canonical parity suite (IT-005 extension point)
- `internal/tools/builtin/skills.go:13-58` + `internal/daemon/native_tool_registry_skills.go:185-250` + `internal/toolmeta/native_entries.go:208-210` — native tool descriptors/handlers/presentation
- `internal/extension/host_api_skills.go:13-78` (`hostAPISkillSummary`, `Source` at `:57`) + `host_api_methods.go:71` + `extension/contract/skills.go` — Host API closure
- `internal/cli/skill_commands.go:21-43` + `skill_commands_mutation.go:20-127` (`where:20-66`, `create:67-127`) — verb registration seams
- `internal/cli/skill.go:12-98` + `skill_output.go` + `skill_items.go` — DTOs + text/JSON rendering
- `internal/cli/skill_managed_guard.go:12` — managed-session guard (expose verbs join the native-tool steering rules)
- `internal/api/contract/status.go:104-110` — `SkillRuntimeStatusPayload` (explicit no-change, guarded by existing tests)
- `internal/skills/observe_events.go` + `internal/events/names.go` — event names the coverage matrix asserts

### Dependent Files

- `web/src/generated/compozy-openapi.d.ts` + `web/src/systems/skill/adapters/skill-api.ts` — task_06 consumes the expose contract
- `packages/site/content/docs/cli/skill/*.mdx` — new verbs regenerate in task_07 (`make codegen` CLI docs)
- `internal/testutil/e2e` fixtures — CLI E2E journeys use `_dx.md` transcripts verbatim

### Related ADRs

- [ADR-011](adrs/adr-011.md) — expose verb + create option, preset targets only · [ADR-013](adrs/adr-013.md) — origin in CLI/lists · [ADR-014](adrs/adr-014.md) — no copy fallback (error contract) · [ADR-015](adrs/adr-015.md) — four-state vocabulary on every surface

### Web/Docs Impact

- `web/`: generated types regenerate (expose routes, `origin`/`exposures` payloads); consumed by `web/src/systems/skill/adapters/skill-api.ts` + `hooks/use-skill-actions.ts` in task_06. No component edits here.
- `packages/site`: none in this task — generated CLI docs for the new verbs + API page updates land in task_07 (`content/docs/cli/skill/`, `content/docs/api/skills.mdx`).
- QA impact: flag (walk deferred to task_09) two new content-addressed `untested` scenarios: (a) expose lifecycle via CLI — create --expose, expose/unexpose, idempotent repeat, failure renderings, info/where inspection across the four states; (b) skill list/info origin attribution across CLI and API for preset + custom sources.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: extension Host API `handleSkillsList` gains `origin` via the shared DTO with `extension/contract/skills.go` updated in the same change; native-tool descriptors + schema digests refreshed. Checked: bridge SDKs (no skill payloads), MCP sidecars, protocol docs — unaffected.
- Agent manageability: this task ships the agent path — expose endpoints + CLI with structured output and the deterministic per-target error vocabulary; agent-scope authorization split implemented and tested; UDS parity proven.
- Config lifecycle: none — no new keys (checked: no `config.toml` surface in this task).

## Deliverables

- Expose/unexpose endpoints (both transports) with the single failure envelope and authorization split
- `origin`/`exposures[]` closed across list route, detail, native tools, extension Host API — digests refreshed
- CLI verb family with `_dx.md`-exact human and JSON renderings, including failure paths and partial success
- Observability coverage-matrix suite guarding the canonical event contract
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there. Owning suites: `internal/api/core/skills_test.go`, `internal/api/spec/spec_test.go`, `internal/api/udsapi/transport_parity_integration_test.go`, native-tool + `internal/extension/host_api_test.go` suites, `internal/cli/skill_test.go` + `skill_daemon_test.go`, e2e runtime lane.

- [ ] UT-060, UT-061, UT-091 — expose success shape, single failure envelope + workspace_id placement, unexpose independence
- [ ] UT-062 — skill detail `origin` + `exposures[]`
- [ ] UT-093 — agent-scope expose allowed vs config write rejected
- [ ] UT-094, UT-095, UT-096 — native tools closure + digests, extension Host API round-trip, list route origin
- [ ] UT-065, UT-066, UT-090, UT-092, UT-098 — CLI outputs: expose/unexpose, create --expose, where golden, info four states, failure renderings + partial success
- [ ] IT-005 — HTTP/UDS parity (settings scope shapes + expose endpoints)
- [ ] IT-014 — observability coverage matrix (events, correlation keys, durable-before-broadcast)
- [ ] E2E-001 — golden path CLI journey (`_dx.md` verbatim)
- [ ] E2E-005 — expose CLI journey (create → link assert → info → unexpose)

## Success Criteria

- Every assigned test case implemented and passing; `make gate` green (task close); `make codegen-check` green
- Every `_dx.md` transcript for these verbs reproduces byte-for-byte in golden tests (E2E-001/005 rely on them verbatim)
- One failure shape only: no code path returns a non-`expose_failed` top-level error for expose/unexpose failures
- Native-tool descriptor/schema digest goldens updated in the same commit batch as the field additions
