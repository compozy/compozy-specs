---
status: completed
title: Per-loop input defaults, migration guide, deep link
type: backend
complexity: high
---

# Task 09: Per-loop input defaults, migration guide, deep link

## Overview

Closes the active P0 upgrade-path slice without the config migrator: `[loops.inputs.<name>]` config sections that supply declared loop inputs when a run omits them, the web run-view deep link printed by `loop run`, and the migration guide delivered on **two complete surfaces** (root `MIGRATION_GUIDE.md` and the docs-site section). The live `compozy migrate config` verb, first-boot legacy-state probe, and in-place upgrade E2E are deferred to task 14.

<critical>
- ALWAYS READ the migration brief, TechSpec, ADRs, and test contract (`_brief.md`, `_techspec.md`, `adrs/`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS: do not implement `compozy migrate config`, retain inert legacy keys, invent a runtime compatibility loader, infer a workspace from CWD, or add an `http.enabled` switch solely for the deep link. Config migrator work belongs to deferred task 14.
</critical>

<requirements>
- MUST add `[loops.inputs.<loop-name>]` sections to global and workspace config supplying values for the loop's **declared** inputs, resolved per key as run > workspace > global > definition default (Safety Invariant 14).
- MUST preserve source provenance before global/workspace config merge so origin is correct for explicit `false`, `0`, and empty valid values; origin MUST never be reconstructed heuristically from the merged `Config`.
- MUST validate definition-contained defaults during static `loop validate`; stored global/workspace defaults are validated only when effective dry-run or run submission resolves the named loop, NOT at config load — loops enroll dynamically. Both phases use typed `input_default` errors `{loop, key, reason}` for unknown keys and type mismatches; configured and per-run unknown keys both fail, so no source has silent-drop behavior.
- MUST implement one input resolver used by `Service.DryRun` and by every persisted start in `internal/loop/service_start.go` before `prepareResolvedStart`. Every declared start kind—manual, CLI, HTTP, UDS, trigger, schedule, webhook, network, extension, and native tool—passes through that service seam; `start_binding.go` owns only allowlist/mapping before delegating to `Service.Start`, and no start surface gets private precedence or validation behavior.
- MUST echo effective inputs with per-key origin (`run|workspace|global|definition`) in dry-run human, JSON, and TOON output, with identical resolution over HTTP and UDS.
- MUST publish effective input values/origins through `LoopPlanPayload`, and publish optional `web_url` through `RunLoopResponse` only when a persisted run was created. Both DTO changes co-ship through OpenAPI, generated TypeScript, HTTP/UDS/native matchers, and CLI JSON/TOON rendering.
- MUST print the web run-view deep link as the last human-readable line of a successful non-dry `loop run` and expose optional `web_url` from the same public response DTO in JSON and TOON, resolving host/port from effective daemon config and pinning public route `/loop-runs/%s` as a shared Go contract constant. Dry-run has no run id and omits both the human link and `web_url`; it never fabricates a route. HTTP host/port are mandatory in the current configuration, so every successful created run has a link; no disabled-HTTP branch is introduced.
- MUST deliver the migration guide on both surfaces with **integral mirrored content** (brief round-7): root `MIGRATION_GUIDE.md` and the site section — neither is a stub or a pointer.
- MUST cover all eight content items in both surfaces: (1) side-by-side old→new command map, (2) config translation table documenting the intended legacy→v0.3 target-or-dropped matrix as the future/manual contract, (3) explicit statement that `compozy migrate config`, backup/idempotency, and first-boot legacy-state probing are **deferred** to task 14 (do not claim a live migrator verb), plus the distinct old `compozy migrate` XML converter note, (4) where the skills went, (5) an explicit **deferred/not-possible** section, (6) the license metadata-correction note, (7) the Go library close-out, (8) the domain outcome (`compozy.com` serves the site from the single cut, `agh.network` retired, legacy docs with `legacy/v0.2` — brief round-11), clean state restart, and `.gitignore` guidance.
- MUST list in the deferred/removed section, each with status and a workaround when one exists: `--parallel-tasks` wave execution, `--multiple`/`--parallel` worktree isolation, TUI cockpit/attach, `reviews fetch/list/show/fix` verbs, the `tasks run` wizard, the recovery agent, the existing legacy review-provider extension abstraction plus all external review fetch/resolve including CodeRabbit (v0.3 review is agent-authored — brief round 12, ADR-008), cost-aware budgets, agent-per-task rules, the external-CLI skill install helper, and the deferred config migrator (task 14).
- MUST include an exhaustive legacy-surface ledger derived from the v0.2.15 CLI tree and public repositories: every old verb/flag family and web route; the old `compozy migrate` XML converter (run it before upgrading when needed; deferred `migrate config` is not its replacement); reusable-agent `setup`; `compozy ext`; public Go facade; Go and TypeScript extension SDKs; `@compozy/create-extension`; and shipped extensions/skills including `cy-capture-decisions`, `cy-idea-factory`, and `cy-qa-workflow`. The public `@compozy/extension-sdk` and `@compozy/create-extension` packages stay frozen on v0.1.x with no v0.3 registry successor; current development uses the private, version-matched repository workspaces. Each row names its exact successor or says removed/deferred with a workaround; no compatibility feature is added.
- MUST describe the license change as a distribution-metadata correction and NEVER as a relicense (ADR-005, amended).
- MUST point every install/upgrade instruction in both guide surfaces at the beta channel (`npm install -g @compozy/cli@beta`, the hosted installer, versioned `go install`), consistent with the Task 10 front door, and MUST NOT offer Homebrew as an install path until the post-beta 0.3.0 stable bump (brief round-11).
- MUST add `scripts/verify-migration-guide-parity.sh`, exposed by `make migration-guide-check`, as the reproducible guide gate. It normalizes frontmatter and site-only MDX markup, compares the eight required section IDs and their canonical content (including the deferred matrix), prints a useful diff, and exits non-zero on drift; task 13 invokes this command as final evidence.
- MUST state in both command maps that `tasks validate` is removed with no false `loop validate` replacement; rewrite bundled process-skill guidance to use only validation the new flow actually provides.
- MUST run `make codegen` and `make codegen-check` because effective-input origin and `web_url` change the public loop plan/run DTO, OpenAPI/TypeScript outputs, native descriptors, and structured renderers.
- MUST place input resolution in a dedicated file and keep it out of `internal/loop/config.go`, which task 06 already splits before this task begins.
</requirements>

## Subtasks

- [x] 9.1 Add dynamic `[loops.inputs.<name>]` config parsing plus source-preserving global/workspace representation; expose it through existing CLI, HTTP/UDS, and native config management surfaces
- [x] 9.2 Implement typed `input_default` validation errors in static definition validation and effective dry-run/submission at their truthful phases, with HTTP/UDS parity
- [x] 9.3 Share input resolution across dry-run and every real-start surface; echo effective inputs with origin in human, JSON, and TOON modes
- [x] 9.4 Publish effective inputs/origins and optional `web_url` through the public loop plan/run DTOs and regenerate the affected contracts
- [x] 9.5 Print the public run-view deep link only for successful created runs, using one optional `web_url` field across JSON/TOON and a pinned route contract constant
- [x] 9.6 Author the root `MIGRATION_GUIDE.md` with all eight content items, marking the config migrator as deferred (task 14)
- [x] 9.7 Author the docs-site migration section with mirrored content, registered in nav/meta and linked from README, changelog, and launch post
- [x] 9.8 Add `scripts/verify-migration-guide-parity.sh` plus `make migration-guide-check`, with content-level normalization/comparison and final-gate invocation
- [x] 9.9 Implement all assigned unit, integration, and E2E cases

## Implementation Details

Follow ADR-007 (input defaults, resolution order, validation timing), TechSpec §Implementation Design > Data Models and API Endpoints, and §Development Sequencing step 4 for the non-migrator slice. Keep input resolution and guide content in separate files under the 500-line cap. Call the shared resolver from `Service.DryRun` and from `Service.Start`/`StartInline` in `service_start.go` before `prepareResolvedStart`; `start_binding.go` remains an allowlist/mapping adapter that delegates to that service path.

Greenfield confirmed by exploration: `packages/site/content/runtime/migration/` does not exist. The deep-link route constant must be shared Go-side because the daemon's SPA index fallback returns 200 for any path — a bare HTTP-200 check would be vacuous. Do not invent a `migrate` CLI verb here; task 14 owns that surface.

The intended config translation/drop matrix remains documented in the guide as the future/manual contract and is implemented by deferred task 14 against the legacy tree at `the CompozyOS repo` (stable v0.2.15 at `8f8908afd70c731b815e20282bacad05aa026827`).

### Relevant Files

- `internal/config/config_load.go`, `internal/config/loops.go`, and merge/resolver paths — new `[loops.inputs.<name>]` sections plus source-preserving global/workspace provenance, type-checked later against the loop contract
- `internal/loop/service.go`, `internal/loop/service_start.go`, `internal/loop/start_binding.go`, `internal/loop/dsl/gate_start.go`, and a dedicated input-resolution file — dry-run plus the common persisted-start seam before `prepareResolvedStart`; `start_binding.go` retains allowlist/mapping only
- `internal/cli/loop.go:215,265-270` and `internal/cli/loop_support.go:381-394` — `loop run`, input/config flag parsing, and the actual `loopOutputBundle` human/structured rendering seam
- `web/src/routes/_app/loop-runs.$runId.tsx` and existing public links to `/loop-runs/$runId` — public path pinned as a Go contract constant
- `extensions/dev-cycle/loops/review-and-fix/loop.yaml` — declared inputs (`task_name`, `reviewer`, `fixer`, `auto_commit` after ADR-008)
- `internal/api/core/settings_*`, `internal/api/contract/settings_*`, `internal/api/udsapi/**`, and native config descriptors — agent-manageable dynamic loop-input defaults
- `internal/api/contract/loops.go`, `openapi/**`, `web/src/generated/**`, and native loop descriptor/catalog outputs — effective-input origin plus optional `web_url` public contract and generated families
- `internal/testutil/e2e/**` and `internal/e2elane/lanes.go` — E2E-001 daemon harness ownership
- `MIGRATION_GUIDE.md` (root, new) and `packages/site/content/runtime/migration/` (new) — the two guide surfaces
- `internal/cli/{root.go,commands.go,commands_simple.go,daemon_commands.go,reviews_exec_daemon.go,runs.go,setup.go,agents_commands.go,workspace_commands.go,validate_tasks.go}` — authoritative legacy verb/flag inventory for the disposition ledger
- `web/src/routes/**`, `extensions/{cy-capture-decisions,cy-idea-factory,cy-qa-workflow}/**`, `sdk/{extension,extension-sdk-ts,create-extension}/**`, and `pkg/compozy/**` — legacy web/extensibility/library surfaces requiring successor-or-no-successor rows

### Dependent Files

- `packages/site/content/runtime/configuration/**` — regenerated config reference including the new sections
- `packages/site/lib/__tests__/public-install-contract.test.ts` — reads the root README, which links the new guide
- `README.md` (root) — links `MIGRATION_GUIDE.md`
- `.gitignore` guidance in the guide — runtime entries inside the partially committed `.compozy/` directory
- `scripts/verify-migration-guide-parity.sh` and `Makefile` — canonical reproducible parity command; task 13 consumes its evidence

### Related ADRs

- [ADR-007: Per-Loop Input Defaults in Configuration (`[loops.inputs.<name>]`)](adrs/adr-007.md) — the new config sections, resolution order, validation timing, and the runtime-selection scope boundary
- [ADR-006: Flat `.compozy/` Namespace and Explicit Config Migration](adrs/adr-006.md) — document the intended migrate-and-drop contract in the guide; live migrator implementation is deferred to task 14

## Deliverables

- `[loops.inputs.<name>]` resolution shared by every start origin, with typed validation errors and per-key origin echo across CLI, HTTP, UDS, JSON, and TOON
- Deep link printed only by a successful non-dry `loop run` and exposed through optional `web_url` in JSON/TOON, against a pinned route contract constant
- Root `MIGRATION_GUIDE.md` and the docs-site migration section, both complete and mirrored, including the deferred/not-possible section and an explicit deferred-migrator note pointing at task 14
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from [`_tests.md`](_tests.md), the test contract — read each ID's full definition there before writing tests.

- [x] UT-056 — deep link as the last human line plus matching `web_url` in JSON/TOON for a successful non-dry run; dry-run omits all three, and the URL derives from effective daemon host/port plus the public pinned route
- [x] UT-063–UT-066 — input defaults: per-key workspace-over-global resolution with false/zero provenance, configured and per-run unknown-key typed errors, type-mismatch rejection pre-spawn, required input satisfied by config with explicit run values always winning
- [x] IT-015 — deep link against a daemon on a random port matches the public pinned route template and resolves
- [x] IT-018 — input defaults through the real submission path with origin reporting and HTTP/UDS parity, including explicit run override and unknown-key validation failure
- [x] E2E-001 — legacy-user delivery journey: seeded task tree, mixed per-task runtimes, run completes, runtimes visible in JSON, deep link printed

### Web/Docs Impact

- `web/`: none beyond the deep-link target — checked surfaces: `web/src/routes/_app/loop-runs.$runId.tsx` and public `/loop-runs/$runId` links; reason: the runtime display shipped in task 06 and this task only links to it. The Go constant is `/loop-runs/%s`, not the TanStack internal `_app` route id.
- `packages/site`: new `content/runtime/migration/` section registered in nav/meta, regenerated `content/runtime/configuration/**` for the new input-default sections, regenerated `content/runtime/cli-reference/**` for dry-run origin output (no live `migrate config` verb), plus README/changelog/launch-post links to the guide.
- QA impact: new scenarios — add content-addressed `untested` files for deep-link output and workspace input defaults; reset scenarios whose `entry_points` cite `loop run` output or config file locations. Do not add migrator scenarios here — those belong to deferred task 14.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: no new extension surfaces — checked manifests, hooks, tools/resources, bundles, registries, bridge SDKs; input defaults are runtime config resolution. Any loop gains defaults without loop-YAML changes.
- Agent manageability: existing config CLI, HTTP/UDS, and native config surfaces can inspect/mutate dynamic loop-input defaults; workspace-scoped loop dry-run echoes effective inputs with origin; validate/submission return typed `input_default` items. No live migrator CLI surface ships in this task.
- Config lifecycle: adds `[loops.inputs.<loop-name>].<input>` in global and workspace config with run > workspace > global > definition precedence, represented before merge to preserve provenance, validated at loop resolution, and documented in regenerated references/guides. Live legacy-config translation is deferred to task 14.

### Compozy Impact Audit

- Native tools: existing config and Loop descriptors/schemas now expose dynamic scalar defaults, `input_origins`, typed `input_default` failures, and optional `web_url`; OpenAPI, generated TypeScript, descriptor matchers, and schema digests were regenerated. No migrator ToolID was added.
- Extensibility and hooks: no impact after checking extension manifests, hooks, bundled resources, registries, bridge SDKs, and MCP sidecars; defaults enter through the core Loop service seam and add no extension-specific resolution path.
- Workspace data isolation: defaults are global- or workspace-scoped config data. Workspace writes resolve `workspace_id` through the canonical workspace resolver; resolver snapshots include both global and workspace config files. IT-018 proves a second workspace sees the global `false` value/origin and never the first workspace's `true` override; existing foreign-run HTTP/UDS/native/SSE isolation remains green.
- Official Compozy skill: `skills/compozy/references/loops.md` documents `[loops.inputs]`, origin precedence, typed validation, the removed `tasks validate` surface, public run URLs, and the deferred Task-14 config migrator.

## Success Criteria

- Every assigned test case implemented and passing
- A delivery run can consume workspace/global `[loops.inputs.*]` defaults with truthful origin echo, and a successful non-dry `loop run` prints a resolvable deep link
- Both guide surfaces carry identical normalized content, verified by the parity command, mark the config migrator as deferred (task 14), and the disposition ledger accounts for every audited legacy CLI/web/extension/SDK surface with an exact successor or removed/deferred status and workaround
- `make codegen-check` diff-clean after public DTO regeneration; `make verify` green
