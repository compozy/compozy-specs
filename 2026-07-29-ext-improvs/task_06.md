---
status: completed
title: Contributed commands surface (ADR-008, fixture-proven)
type: backend
complexity: high
---

# Task 6: Contributed commands surface (ADR-008, fixture-proven)

## Overview

Ship the extension-contributed command surface as pure presentation over the existing tool runtime: command/group/flag specs in the generated contract, build-time validation with a closed flag-projection subset, a storage-free read model, `extension commands` discovery and `extension exec` two-phase invocation — proven by an E2E fixture extension. **No product command ships**: the dev-cycle `archive` port belongs to the `cmd-archive` follow-up program.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST add `ExtensionCommandSpec`, `ExtensionCommandGroupSpec`, `CommandFlag`, and `CommandFlagType` to `internal/extension/contract/command.go` exactly as the TechSpec defines them, generated into both SDKs (`ext.commandGroup()` TS + Go equivalent) and byte-checked by `make codegen-check`.
- MUST carry declared groups through the full generated path: `DescribePayload.command_groups` → manifest `[[resources.command_groups]]` (+ `command` block on `ToolConfig`) → loader → read model.
- MUST validate closed at build AND manifest load (invariant 17): `/`-joined paths, depth ≤2, uniqueness per extension, groups non-executable/single-segment/leaf-backed, reserved host flags rejected (`cmd`,`input`,`output`,`o`,`json`,`help`), every flag resolving to a top-level schema field, and the closed projection subset (scalars, nullable scalars, enums of scalars, arrays of one scalar post-`$ref`; `object`/composed/tuple/multi-type/unresolvable rejected with `--input` remediation).
- MUST implement `Manager.Commands(workspaceID)` as a projection over active tool descriptors + manifest groups (no new storage): flags in canonical lexicographic order, `ApprovalRequired` as static approval metadata, workspace-filtered for agent callers, disabled/unavailable extensions contributing nothing.
- MUST expose `GET /api/extensions/commands` (`?extension=` filter) with UDS parity returning leaves (verb, tool id, summary, `CommandFlag[]`, risk class, `approval_required`) + groups — CLI/HTTP/UDS is the complete MVP discovery surface (no web list).
- MUST implement `extension commands` (human tree / structured flat array) and `extension exec <ext> --cmd <verb-path>` with two-phase parse: host flags → descriptor-projected flags; argv→typed input document (per-flag conversion errors naming flag + field; repeated flags append; booleans presence-true/`=false`; absent optional omits — the CLI never injects defaults; `--input` mutually exclusive with projected flags); then exactly one `POST /api/tools/ext__<ext>__<tool>/invoke` — policy, approvals, risk gates, `trusted_workspace` unchanged (invariant 16); result through the standard output bundle; unknown paths print available leaves + did-you-mean.
- MUST ship the command-declaring E2E fixture extension (`internal/extension/testdata`-backed: flat `greet` leaf + declared `review` group with `review/fetch` leaf + one approval-gated command) proving the surface on a stamped binary.
</requirements>

## Subtasks

- [x] 6.1 Contract types (`command.go`) + SDK generation + `ext.commandGroup` ergonomics in both SDKs
- [x] 6.2 Manifest carriage (`command` block + `[[resources.command_groups]]`) + build/load validation matrices (paths, groups, flags, schema subset)
- [x] 6.3 `internal/extension/command.go` read model (projection, lexicographic flags, approval metadata, workspace filter)
- [x] 6.4 Route + UDS parity for `GET /api/extensions/commands`
- [x] 6.5 CLI `extension commands` (tree + structured) and `extension exec` (two-phase parse, conversions, single-invoke, bundle rendering, did-you-mean)
- [x] 6.6 E2E fixture extension with nested/flat/approval-gated commands
- [x] 6.7 Implement every assigned test case

## Implementation Details

TechSpec: Core Interfaces (commands block incl. `CommandFlag`), API Endpoints (commands row), Safety Invariants 16–17, Extensibility (Commands bullet). Phase F (commands portion). Two round trips per exec (descriptor fetch + invoke) are accepted (ADR-008 Consequences).

### Relevant Files

- `internal/extension/contract/{command.go,sdk.go}` — new authoritative specs beside the describe schema
- `internal/extension/build.go` (task_03) — build-time validation extension point
- `internal/extension/command.go` — new read model (suite: new `command_test.go`)
- `internal/api/core/extensions.go` — commands handler + parity (suite: `extensions_test.go` for IT-018)
- `internal/cli/{extension_exec.go,extension_commands.go}` — new verbs (suites: `extension_exec_test.go`, `extension_exec_integration_test.go`)
- `internal/tools/provider_descriptor.go` — `ExtensionToolRuntimeDescriptor` gains the optional `command` block (same category as `friendly_verb`/`preview`)
- `internal/daemon/extension_commands_integration_test.go` — new IT-020/021 suite

### Dependent Files

- `sdk/typescript/src/*` + `sdk/go/*` — registration ergonomics + describe payload additions
- `openapi/compozy.json` + generated TS — payload co-ship (L-007)

### Related ADRs

- [ADR-008: Extension-contributed commands](adrs/adr-008.md) — primary (fixture-proven; archive excluded)
- [ADR-003: Uniform SDK architecture](adrs/adr-003.md) — generated contract path
- [ADR-006: Closed-surface positioning](adrs/adr-006.md) — supersession note for the CLI item

### Competitor References

- `.resources/claude-code/utils/plugins/schemas.ts:906-1000` — commands as declarative metadata on plugin manifests
- `.resources/pi/packages/coding-agent/src/core/extensions/types.ts` — `registerCommand` authoring shape

## Web/Docs Impact

none in `web/` by decision (R3 B-008 fork): CLI/HTTP/UDS discovery is the complete MVP surface — a web command list is deliberately not promised. Contributed-commands guide (declaring commands/groups, flag subset, `exec` vs `tool invoke`) owned by task_09. Generated types refresh here.

**QA impact**: add content-addressed `untested` scenarios for command discovery + exec (tree rendering, projected-flag invocation, group error, `--input` escape, approval-gated command); no existing scenario resets (surface is new).

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: extensions gain the command presentation surface (metadata on tools + declarable groups); no new capability, protocol method, registry, or authority — the tool runtime is reused whole (L-031).
- Agent Manageability: agents keep invoking `ext__*` tools natively (no new tool); `extension commands`/`exec` carry `-o json|jsonl|toon`; deterministic unknown-path/conversion/mutual-exclusion errors.
- Config Lifecycle: none — checked surfaces: no `config.toml` key; reserved-flag set is a versioned contract list in the spec, not config.

## Deliverables

- The complete fixture-proven command surface (contract → manifest → read model → route/UDS → CLI)
- Closed validation: malformed trees/flags/schemas fail at build and load, never at runtime
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] UT-071, UT-072, UT-073, UT-074, UT-075 — projection (full `CommandFlag` shape, lexicographic), reserved flags, missing field, duplicate verbs, depth
- [x] UT-076, UT-077, UT-078, UT-079 — argv→JSON conversions, typed errors, `--input` exclusivity, group non-executable + unknown-path listing
- [x] UT-080, UT-081 — read model determinism + disabled/unavailable exclusion; human tree + structured golden
- [x] UT-082, UT-083 — group validation matrix; groups round-trip describe→manifest→read model (+ implicit prefix nodes)
- [x] UT-084, UT-085 — conversion-semantics matrix; rejected-subset matrix with `--input` remediation
- [x] IT-018 — commands read-model HTTP/UDS parity, workspace-filtered, `CommandFlag` contract shape
- [x] IT-019 — exec performs exactly one invoke (transport spy) across output formats
- [x] IT-020 — policy/approval parity: exec ≡ tool invoke; unavailable-tool reason codes
- [x] IT-021 — nested exec: leaf invokes; group errors with zero runtime calls
- [x] E2E-008 — fixture journey on a stamped binary (discovery, exec, group error, approval gate)

Contract co-ship gates (L-007): `make codegen && make codegen-check` after the commands payload addition + `bunx turbo run typecheck test --filter=./web` (regenerated types compile).

## Success Criteria

- Every assigned test case implemented and passing
- `extension exec` can never bypass a gate the equivalent `tool invoke` enforces (IT-020 green)
- A malformed command tree is unrepresentable at runtime (build + load both reject every matrix row)
- Zero product commands in-repo; fixture proves the surface end to end

## Completion Notes

Implemented contributed commands as a storage-free presentation layer over the existing tool runtime.
Both SDKs now author tool command metadata and presentation-only command groups; describe mode carries
them into generated manifests; build and load reject malformed trees, reserved flags, and schema shapes
outside the closed projection subset. The read model exposes only active, healthy commands for the
effective workspace through shared HTTP/UDS handlers. The CLI discovers trees and performs two-phase
execution with typed projected flags or schema-validated `--input`, then calls the canonical tool invoke
path exactly once with workspace, session, agent, and approval context unchanged.

The stamped Go fixture declares flat, nested, and approval-gated leaves. Its monotonic response counter
proves group errors and denied policy calls never reach the runtime. Approval and backend-unavailable
diagnostics are byte-equivalent to `tool invoke`; unavailable/conflict reasons take precedence over
static approval metadata, while approved calls consume normal single-use tool approval tokens.

Library research used Exa plus authoritative package documentation. The implementation retains Cobra/
pflag for runtime-discovered argv, `santhosh-tekuri/jsonschema/v6` for schema compilation, `$ref`
resolution, and raw-input validation, and Jennifer for the existing generated Go contract path. No
maintained runtime JSON-Schema-to-Cobra library fit the descriptor model without introducing a second
static struct/tag DSL, so no dependency was added.

Fresh focused evidence after source freeze:

- `make lint` — PASS: zero Go issues, zero Bun warnings, Oxfmt clean, source policies and the 500-line cap clean.
- Affected Go packages — 5,029 race tests PASS across tools, extension, API, CLI, and daemon.
- Tagged IT-019/020/021 + E2E-008 — 10 tests PASS across CLI and daemon.
- Go SDK — 68 race tests PASS across three packages.
- TypeScript SDK — lint/typecheck/codegen plus 54 tests PASS; all generated modules and 13 manifests verified.
- Web contract consumer lane — 5/5 tasks PASS; 515 files and 4,046 tests PASS.
- QA tracker — 668 scenarios materialized; five new command scenarios are `untested` under `J-run-extension-commands`.

The single program-wide `make verify` remains deferred until all eleven tasks are complete, per the
accepted loop execution contract.

Compozy Impact Audit:

- Native tools: no new `compozy__*` ID, toolset, native input schema, digest, or capability gate. Checked the central tool registry plus `tool invoke`/`tool approve`: `extension exec` resolves presentation metadata and invokes the same `ext__*` tool ID through the same approval, availability, risk, and reason-code paths.
- Extensibility and hooks: added generated SDK command/group authoring, describe/manifest carriage, active-descriptor projection, API discovery, and CLI presentation. No new capability, extension protocol method, hook event, resource family, bundle, registry source, bridge SDK, MCP sidecar, or `config.toml` key/default was added; those exact surfaces were checked and remain unchanged.
- Workspace data isolation: command descriptors are derived data with no store, cache, event, or new datum. Published commands are global; dev commands resolve through the trusted workspace actor and server-side instance selection. HTTP/UDS parity and scoped integration coverage prove foreign workspace commands do not leak; exec forwards workspace/session/agent scope unchanged to the canonical invoke path.
- Official Compozy skill: public command authoring and `extension commands|exec` guidance is an identified impact owned by Task 09's contributed-commands documentation/official-skill pass; Task 06 co-ships the runtime contract and records that downstream requirement without duplicating the guide early.

Web/Docs Impact:

- `web/`: no command-list UI by the accepted MVP decision. OpenAPI and generated Web types were refreshed, and the full Web Turbo lane proves consumers compile and test against the new payload.
- `packages/site`: no guide changed; Task 09 owns the contributed-commands guide and worked example.
- QA: added five content-addressed `untested` scenarios plus their CLI/API journey. No existing verdict was reset because the surface is new.
- Visual verification: not applicable — no Web or shared-UI component changed.
