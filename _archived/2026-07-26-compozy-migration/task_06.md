---
status: completed
title: Per-task runtime selection — engine, binder, contract, CLI, web
type: backend
complexity: high
---

# Task 06: Per-task runtime selection — engine, binder, contract, CLI, web

## Overview

The headline P0 parity capability (Linear AGH-80): a `runtime {provider, model, reasoning}` object that merges field-by-field across per-run rules, task frontmatter, config rules, node params, and loop defaults — resolved by the loop engine, applied by the daemon policy gate, validated before any session spawns, and reported truthfully in run status and the web run view. Delivered as one vertical slice because the contract DTOs, generated types, and consuming surfaces must co-ship with the engine change.

<critical>
- ALWAYS READ the migration brief, TechSpec, ADRs, and test contract (`_brief.md`, `_techspec.md`, `adrs/`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS: remove retired runtime keys at every ingress and fail with the migration-guide diagnostic; do not retain aliases, compatibility fallbacks, or silently ignored fields.
</critical>

<requirements>
- MUST implement field-level merge with origin-ordered precedence: per-run rules > task frontmatter `runtime` > config-layer rules > node `params.runtime` > `runtime_defaults` > agent definition (Safety Invariant 1).
- MUST keep per-run rules separated from config rules in `EffectiveConfig` so frontmatter can sit between them.
- MUST implement selector specificity with later rules winning only at equal specificity (Safety Invariant 2).
- MUST preserve legacy complexity defaults as `complexity` runtime rules and resolve specificity as id > type > complexity, with later rules winning only within the same specificity. Persisted legacy `tasks.run.task_runtime_rules` are type-only; `[defaults.by_complexity.{low,medium,high,critical}]` is the distinct complexity source and MUST not be widened into a different selector.
- MUST resolve precedence in `internal/loop` and apply the triple in the daemon policy gate — the engine MUST NOT resolve provider credentials, auth mode, home, or env policy (Safety Invariant 4, L-005).
- MUST build the `RuntimeCatalog` validation surface without inventing catalog authority. Provider IDs always validate after aliases canonicalize through `CanonicalProviderName` at `internal/config/provider_builtin.go:234`. Model membership validates only when `modelcatalog.HasAuthoritativeProviderCatalog(provider)` is true, using the existing offline-safe/no-refresh session-preflight semantics; providers without an authoritative model catalog MUST NOT have arbitrary model IDs falsely rejected. The interface is defined in `internal/loop` where it is consumed and implemented against `internal/modelcatalog`; resolved status reports canonical provider IDs and the submitted model ID.
- MUST split validation truthfully: `loop validate` validates only definition-contained static runtime syntax because its current request contains a definition file; full effective-layer validation happens through `loop run --dry-run` and submission, then again per item before bind. Both paths return deterministic `runtime_validation` items `{task_id?, field, value, reason}` and never spawn a session on failure (Safety Invariant 3).
- MUST return `AppliedRuntime` on the binding result copied from the final `CreateOpts`, and MUST source all run-status/event runtime values from it — engine-empty fields report `source=agent` (Safety Invariant 5). Every durable/read surface remains workspace-isolated under Safety Invariant 13.
- MUST enforce `bindingMismatch` when a bind request's triple diverges from a pinned creation profile (Safety Invariant 4).
- MUST extend `ResolveSessionAgentWithRuntime` at `internal/config/provider_resolve.go:219` with a reasoning override; that file, not `provider.go`, owns the current resolver.
- MUST delete `model_defaults` / `LoopModelDefaults` across all three DTO layers, `dsl.Contract.ModelDefaults`, `EffectiveConfig.ModelDefaults`, `WorkerModel`/`JudgeModel` action inputs, `ActionSessionBindRequest.Model`, `dsl.RunAgentParams.Model`, and every judge-model surface: `dsl.GateCriterion.Model`, gate `EvaluateInput.JudgeModel`, `JudgeRequest.Model`, and evaluator fallback plumbing. The owning config/loop delete targets include `internal/config/{loops.go,merge_loops.go,tool_surface_loops.go}`, `internal/daemon/{loop_defaults.go,loop_goal_executor.go,loop_goal_command_definition.go}`, `internal/loop/{service_types.go,config.go}`, `internal/loop/dsl/gate_start.go`, and `internal/loop/gate/**`. This is a hard cut: no aliases; strict decode rejects old keys with an error citing the migration guide.
- MUST implement the repeatable `--runtime` flag with slash-safe grammar `provider/model[@reasoning]` (`-` skips a field; bare value is model-only sugar; invalid reasoning suffix is a parse error, never absorbed into the model; rule match values may not contain `:`).
- MUST parse task frontmatter `runtime:` and `type:` in the dev-cycle import parser and expose them as `.item.runtime` / `.item.type`. Strict known-field decoding applies to the nested `runtime` object only; it MUST NOT reject unrelated existing task frontmatter such as `complexity`.
- MUST co-ship those frontmatter fields through the import tool output schema, manifest descriptor, generated digest, and embed contract; parsing them without publishing the payload is incomplete.
- MUST give judges `runtime_defaults.judge` plus per-criterion `runtime`, with NO task-rule matching (Safety Invariant / ADR-001 §7); sub-loops resolve their own layers with no implicit parent propagation.
- MUST render resolved runtime per task in the web run view as read-only display with provenance — no invented controls (SD-007).
- MUST route that visible web display through the `designer` agent in execution mode, activate `eng-design`, `ui-craft`, and `impeccable` before editing, and capture `eng-ui-screenshot` evidence before completion.
- MUST run the complete affected `make codegen` graph and co-ship every changed generated family: Atlas/sqlc, Daytona sidecars (always generated by `Codegen`), `compozy-codegen all` (`openapi`, `sdk-contracts`, `loop-enums`, `lifecycle-matrix`, `native-tool-catalog`), generated TypeScript OpenAPI declarations, and the DESIGN sync. `CodegenCheck` alone may skip the Daytona build/check through its unchanged-input stamp; `make codegen-check` owns drift, and no fixed artifact count is assumed.
- MUST persist `resolved_runtime` with field provenance per generation output/node item, not only in ephemeral action state, so post-restart run status, HTTP, UDS, SSE, CLI, web, and native status tools all derive the same AppliedRuntime truth. This is a schema change and follows the append-only schema/Goose/sqlc process.
- MUST reject every retired runtime key at every input boundary: `model_defaults`, node `params.model`, and retired criterion/model keys. `NodeParams` is map-shaped and permissive decoding would otherwise silently retain `params.model`; errors MUST name the retired key and migration-guide reference.
- MUST expose runtime configuration through the existing config management surfaces (CLI, HTTP/UDS, and native config tools) rather than adding a configuration that agents cannot inspect or mutate.
- MUST split `internal/loop/config.go` before adding runtime behavior: it is already close enough to the 500-line production cap that both this task and task 09 cannot safely grow it. Runtime contracts/merge/resolution/validation land in responsibility-named files; task 09's input resolver lands in its own file and reuses the shared start path.
</requirements>

## Subtasks

- [x] 6.1 Add the runtime shapes to the DSL (spec, rule, match) and replace the node-level `Model` with `params.runtime`
- [x] 6.2 Extend loop config merge with runtime defaults/rules across all four layers, preserving per-run vs config origin
- [x] 6.3 Implement per-item resolution with per-field provenance in a new resolver file
- [x] 6.4 Build the runtime validation surface (interface in the engine, implementation over modelcatalog, alias canonicalization) and wire static validation to `loop validate`, effective validation to dry-run/submission, and final validation to pre-bind
- [x] 6.5 Carry the resolved triple through the bind request; apply it in the policy gate with the reasoning override; return `AppliedRuntime`
- [x] 6.6 Enforce isolated/ephemeral binding when triples diverge from pinned profiles
- [x] 6.7 Add judge runtime defaults and per-criterion runtime to the gate evaluator
- [x] 6.8 Parse `runtime`/`type` task frontmatter and expose them to the graph
- [x] 6.9 Replace the contract DTOs, delete every `model_defaults`/judge-model surface, and regenerate the full affected codegen graph
- [x] 6.10 Implement the `--runtime` flag parser and the dry-run echo of parsed layers
- [x] 6.11 Add the append-only persistence/store projection for per-output `resolved_runtime`, then project it with provenance into run status, SSE, CLI, HTTP, UDS, web, and native status payloads
- [x] 6.12 Render read-only resolved runtime with provenance in the web run view
- [x] 6.13 Implement all assigned unit, integration, and E2E cases

## Implementation Details

Follow TechSpec §Implementation Design > Core Interfaces for the runtime boundary, §Safety Invariants 1-5 and 13, and §Development Sequencing step 2. Activate `eng-schema-migration` before changing the durable output projection: update the declarative schema, append the next Goose migration, refresh sqlc/atlas artifacts, and extend the canonical fresh/reopen/integrity suites. The resolver lands in a new file; keep every new file under the 500-line cap and split by responsibility (resolution, validation, flag parsing) rather than growing existing files.

Greenfield display surfaces confirmed by exploration: the CLI has no model flags today (`internal/cli/loop.go:265-270`), and the web run view renders no model/runtime. The existing `model_defaults` contract is not greenfield: it appears in runtime config, loop service/daemon plumbing, judge evaluation, generated contracts, MSW fixtures, and a session-goal transport error. Delete all of those existing surfaces while adding only the already-approved read-only display.

### Relevant Files

- `internal/loop/dsl/node_params.go:4-13` — `RunAgentParams` with `Model string` at :9 (delete target)
- `internal/loop/dsl/contract.go:15,53-57` — `Contract.ModelDefaults` and the `ModelDefaults` struct (delete targets)
- `internal/api/contract/loops.go:188,191-195,209,212-216,387,401,447` — three DTO layers carrying model defaults plus the node-params payload `Model`
- `internal/loop/config.go:92,104-124,170,208` — four-layer merge, `modelDefaultsLayer`, `mergeConfigLayer`; extract responsibility-named runtime files before editing so this file does not approach the cap
- `internal/config/{loops.go,merge_loops.go,tool_surface_loops.go}`, `internal/daemon/{loop_defaults.go,loop_goal_executor.go,loop_goal_command_definition.go}`, and `internal/loop/service_types.go` — existing `model_defaults` config/default/plumbing delete targets
- `internal/loop/generation_snapshot.go:22-29`, `internal/store/globaldb/schema/definitions/50_loops.sql:46-56`, and the next append-only Goose migration/sqlc output — durable per-output runtime provenance; schema source and migration must co-ship
- `internal/loop/action_runagent.go:61,102-107` — `runAgentModel(spec.Model, in.WorkerModel)` resolution to replace
- `internal/loop/action_types.go:80-81,183,192-193,220,252` — action input worker/judge model, bind request, `BindActionSession`
- `internal/loop/coordinator_action.go:213,247-248` — the only effective→action model plumbing site
- `internal/loop/dsl/gate_start.go`, `internal/loop/gate/types.go`, and `internal/loop/gate/evaluator_judge_human.go:47-59` — criterion/evaluate/request judge-model fields and fallback plumbing to replace with runtime shapes
- `internal/daemon/loop_managed_binding_profile.go:17,63,70,90,97,118` — creation profile and `baseCreateOptions`
- `internal/daemon/loop_action_policy.go:19,39,61,82-84` — policy gate; note :82 currently overwrites the request model with the resolved-agent model
- `internal/config/provider_resolve.go:214,219` — `ResolveSessionAgent` / `ResolveSessionAgentWithRuntime`
- `internal/config/provider_builtin.go:234` — `CanonicalProviderName` (alias → stable provider id)
- `internal/modelcatalog/{service.go:80,authoritative_provider_models.go:14-21,reasoning.go:10,48,53,61}` — catalog read path, cursor-only allowlist, reasoning enum
- `internal/session/{runtime_overrides.go:13-38,runtime_model_preflight.go:20}` — existing effort validation and preflight
- `internal/cli/loop.go:215,265-270` + `loop_support.go:164,182,229` — `loop run` and its flag parsing
- `extensions/dev-cycle/import_tasks_parser.go`, `schemas.go`, `extension.json`, and `embed_test.go` — frontmatter parse plus import-task output schema, descriptor digest, and embedded contract
- `internal/api/contract/loops.go`, `internal/daemon/loop_api_payloads.go`, `internal/api/udsapi/**`, and native loop tool schemas — run-status/runtime projection across every public transport
- `internal/cli/config*.go`, `internal/api/core/settings_*`, and native config descriptors — agent-manageable runtime configuration lifecycle
- `web/src/systems/loops/components/run-page/**` — real run UI (the route file is a 13-line shell)
- `magefiles/codegen.go:12-35`, `cmd/compozy-codegen/main.go`, and `magefiles/defaults.go:24-27` after task 01 — the complete generated graph and its output paths, including lifecycle and native-tool catalogs

### Dependent Files

- `web/src/generated/{agh,compozy}-openapi.d.ts` — regenerate from the contract change
- `web/src/systems/loops/mocks/{fixtures.ts:513,handlers.ts:282}` — MSW fixtures carrying `model_defaults`
- `web/src/systems/session/lib/session-goal-chat-transport.ts` — user-facing validation text that currently names `loops.defaults.delivery.model_defaults.judge`
- `web/e2e/**` — E2E-004 harness and real run-view coverage
- `extensions/dev-cycle/loops/*/loop.yaml` — audit confirms no shipped dev-cycle loop currently uses `params.model`; retain hard-cut validation coverage rather than inventing a YAML migration
- `internal/loop/gate/**` existing evaluator tests — extended for the criterion rename

### Related ADRs

- [ADR-001: Per-Task Runtime Selection — Field-Merged `runtime`, Engine-Resolved, Binder-Applied](adrs/adr-001.md) — the field-merge model, resolution/application split, validation, and scope boundaries this task implements
- [ADR-002: Per-Invocation Runtime Override Through Dedicated `--runtime`](adrs/adr-002.md) — flag grammar, its slash-safe parsing rules, and the divergence from the brief's `--input` wording

## Deliverables

- Field-merged per-task runtime resolution with per-field provenance and preflight validation
- Policy-gate application returning `AppliedRuntime`, with isolated-bind enforcement on divergence
- Judge runtime defaults plus per-criterion runtime, no task-rule matching
- Replaced contract DTOs with every `model_defaults` and retired judge-model surface deleted and the full affected codegen graph regenerated
- Repeatable `--runtime` flag with dry-run layer echo, plus `resolved_runtime` in run status/SSE
- Read-only per-task runtime display with provenance in the web run view
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from [`_tests.md`](_tests.md), the test contract — read each ID's full definition there before writing tests.

- [x] UT-001–UT-009, UT-060, UT-061 — resolution engine: layer precedence, field merge, id/type/complexity specificity, non-task items, sub-loop boundary, node-params layer ordering
- [x] UT-010–UT-013, UT-062 — validation: unknown provider/model, empty match, bad reasoning, empty triple, alias canonicalization and gateway-style slash model ids
- [x] UT-014–UT-016 — task frontmatter parsing: present, absent, malformed
- [x] UT-017–UT-019 — binder application: provider re-derivation, policy failure, `CreateOpts` copy and `AppliedRuntime` equality
- [x] UT-020–UT-022 — judge runtime: per-criterion override, judge validation path, task rules never applying to judges
- [x] UT-023–UT-027 — `--runtime` grammar: full triples, rule forms, slash-safe models and `-` skips, parse errors, repeat ordering
- [x] UT-028 — run-status projection with per-field provenance including `node` and `agent` sources
- [x] UT-029–UT-032 — config layers: merge order and origin, watch/delivery classification, `model_defaults` rejection, criterion key replacement
- [x] IT-001, IT-002 — end-to-end loop run with mixed sources against `acpmock`; isolated bind and `bindingMismatch`
- [x] IT-003, IT-004 — static validate endpoint plus effective dry-run/submission validation (HTTP + UDS parity) with 422 body items; provider policy failure spawning no process
- [x] IT-005 — gate evaluation with a criterion-level judge model
- [x] IT-006 — CLI dry-run JSON echo equals the HTTP-submitted effective config
- [x] IT-007 — append-only schema migration plus canonical fresh/reopen/equivalence coverage; durable run status, SSE, CLI, HTTP/UDS, and `compozy__loop_status` match `acpmock` and preserve workspace isolation after reopen
- [x] IT-008 — real loader merge order across config layers; `model_defaults` fails load
- [x] E2E-004 — web run view renders resolved runtime per task with provenance and no runtime controls

### Web/Docs Impact

- `web/`: `web/src/systems/loops/components/run-page/**` (new read-only runtime display with provenance tooltip), `web/src/systems/loops/mocks/{fixtures.ts,handlers.ts}` (drop `model_defaults`, add `resolved_runtime`), `web/src/generated/**` (regenerated), plus any loops query-options/hooks touching run-status payloads. Activate `eng-data-boundaries` for the run-status read path and `eng-ui-screenshot` to verify the rendered display.
- `packages/site`: regenerated `content/runtime/cli-reference/**` for the new `--runtime` flag and regenerated `content/runtime/api-reference/**` for the changed payloads; authored config-reference examples for `runtime_defaults`/`runtime_rules`; `skills/compozy/references/loops.md` documents per-task runtime selection.
- QA impact: new scenarios — add content-addressed `untested` files for `--runtime` invocation, per-task runtime display in the run view, and preflight validation errors; reset any existing scenario whose `entry_points` cite `loop run` flags or run-status payloads.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: `extensions/dev-cycle/import_tasks_parser.go` gains frontmatter fields and its output schema/digest; task 07 consumes that already-published payload. Native loop tools (`compozy__loop_run`, `compozy__loop_status`, `compozy__loop_validate`) carry updated input/output schemas with regenerated digests. No new hook events; no bundle/registry semantics change.
- Agent manageability: `loop run --workspace <ref> --name <name> --runtime ... --dry-run -o json` echoes effective layers; `loop validate --workspace <ref> --file <path>` owns static authored layers; `loop status --workspace <ref> --run-id <id> -o json` exposes durable `resolved_runtime` with provenance. Submission and HTTP/UDS/native surfaces retain parity — no CLI-only state. Deterministic error contract `{task_id?, field, value, reason}`.
- Config lifecycle: adds `runtime_defaults.worker|judge.{provider,model,reasoning}` and `runtime_rules[]` across all four loop-config layers (definition contract, `[loops.defaults.delivery|watch]`, stored per-loop config, per-run overrides); removes `model_defaults.worker|judge` everywhere with strict-decode rejection; config load performs structural decode only, while enum/catalog validation runs in static validate and effective submission/pre-bind paths; examples ship in the config reference and the official skill's loops doc; the legacy translation story belongs to task 09.

### Compozy Impact Audit

- Native tools: update `compozy__loop_run`, `compozy__loop_status`, and `compozy__loop_validate` descriptors, schemas, digests, validation diagnostics, and transport-parity tests for runtime input and durable status output.
- Extensibility and hooks: import-task frontmatter is published through the dev-cycle descriptor/schema/digest; no hook, bundle, registry, bridge-SDK, or MCP-sidecar behavior changes.
- Workspace data isolation: `resolved_runtime` is generation-output scoped under the owning run/workspace; store, API, CLI, native tools, SSE, cache, and event reads must preserve workspace filtering and prove cross-workspace non-leakage.
- Official Compozy skill: update `skills/compozy/references/loops.md` for runtime selection, static versus effective validation, and status/read paths.

## Success Criteria

- Every assigned test case implemented and passing
- `make codegen-check` diff-clean after regenerating every affected family; `make verify` green
- Zero live-schema, production, generated, fixture, or web-copy uses of `model_defaults`, `ActionSessionBindRequest.Model`, `RunAgentParams.Model`, criterion `Model`, `EvaluateInput.JudgeModel`, or `JudgeRequest.Model` remain; only intentional migration-guide/delete-history mentions are allowed and are classified by the grep evidence
- A single loop run resolves different providers per task, each session receives the expected `CreateOpts{Provider, Model, ReasoningEffort}`, and every resolved value in run status traces to `AppliedRuntime`
- An invalid definition-contained provider or authoritative-catalog model fails static `loop validate`; an invalid stored or per-run effective value fails dry-run/submission; non-authoritative providers do not receive false unknown-model errors, and neither failure path spawns an ACP process
