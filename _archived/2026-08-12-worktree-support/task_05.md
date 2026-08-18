---
status: completed
title: Assisted exit, forge.provider surface, bundled GitHub extension
type: backend
complexity: high
---

# Task 5: Assisted exit, forge.provider surface, bundled GitHub extension

## Overview

Delivers the exit engine and the forge extensibility surface: daemon-computed exit plans (ladder + blocked reasons + cleanup evidence), durable exit operations (commit with named untracked scope, push, PR) with op-id compare-and-cancel and restart recovery, the `forge.provider` provide surface (capabilities vocabulary/draft/compare-template, status, idempotent PR create) with consumer registry and conformance coverage, the bundled GitHub extension (secret binding + opportunistic `gh`), the core URL-shape browser fallback, exit routes/CLI verbs, and the program-wide transport parity matrix.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST compute the exit plan daemon-side from status + forge cache + bound-session activity: auto-advancing primary (`commit → commit&push → push → open_pr → view_pr`), per-action `{enabled, blocked_reason}` with the design set's copy literals, global pauses (session running, status unreadable), diverged/behind blocks with no auto-resolution, two-tier cleanup evidence with the merged-precedence rules.
2. MUST run exit actions as durable `worktree_exit_ops` (single-active via partial index), detached lifetime, `CancelExitAction(workspace, worktree, op_id)` compare-and-cancel, boot marking interrupted ops failed, process-group supervision, per-step durable progress events with skip reasons and exactly one success CTA.
3. MUST implement commit per the B-011 decision: stage-all with scope shown first — counts/± plus untracked additions **listed by name**; gitignored files never staged; empty message ⇒ `Update N files`; no daemon text generation (agent-staged path only). Push publishes with `-u` when no upstream. `open_pr` resolves base via `gh-merge-base → upstream → repo default`, prefills an unambiguous template from the base tree, calls the forge idempotently (`opened_existing`), or returns the browser tier.
4. MUST add `forge.provider` as the seventh closed-set provide with service methods `forge/capabilities` (per-remote vocabulary, `supports_draft`, `compare_url_template`, template paths, `credential_source`; deterministic multi-extension winner), `forge/status` (normalized `open|closed|merged` + merged evidence + fetched-at), `forge/pr_create` (idempotent) — consumer registry, manager wiring, conformance-matrix coverage, and machine-readable error causes (`rate_limited`, `credential_expired`, `unsupported_remote`).
5. MUST ship `extensions/forge/github` via the dev-cycle bundled pattern (`embed.go`, `install.go`, manifest with `requires_env=["GITHUB_TOKEN"]`, `__internal extension-provider forge-github` case, `EnsureManagedInstall` at boot): github.com only, credential ladder binding → `gh auth token` → absent, explicit-timeout HTTP client, no MCP sidecar.
6. MUST implement the core URL-shape table (github.com/gitlab.com/bitbucket.org compare links) used only when no extension serves the remote; unknown host ⇒ remote web URL; an extension's `compare_url_template` always wins.
7. MUST expose exit surfaces end-to-end: `GET .../exit`, `POST .../exit/actions`, `POST .../exit/cancel {op_id}` on HTTP+UDS; CLI `compozy worktree exit|commit|push|pr|exit-cancel`; status refresh triggers (post-action, bound-turn end); `?forge=true` explicit forge refresh writing `worktree_forge_status` with staleness.
8. MUST land the full transport parity matrix (IT-033) covering every worktree route and error code now that all routes exist; `make codegen-check` gates the contract artifacts.
9. MUST keep forge credentials process-local to the extension (never in logs/payloads/events) and mark all forge-derived data with `fetched_at` staleness.
</requirements>

## Subtasks

- [x] 5.1 Exit plan computation + copy-literal blocked reasons + cleanup evidence tiers
- [x] 5.2 Durable exit ops + cancel + boot recovery + progress events on the per-worktree stream
- [x] 5.3 Commit/push actions with named-untracked scope + hook-failure surfacing + skip reasons
- [x] 5.4 PR action: base chain, template prefill, forge call, browser tier + URL-shape table
- [x] 5.5 `forge.provider` protocol (capabilities/status/pr_create) + consumer registry + conformance matrix
- [x] 5.6 Bundled GitHub extension (manifest, subprocess, credential ladder, API client)
- [x] 5.7 Exit routes HTTP/UDS + CLI verbs + codegen co-ship
- [x] 5.8 Full transport parity matrix (IT-033) across every worktree route/error code
- [x] 5.9 E2E-001 journey (create → bound session works → commit/push via API → clean removal)

## Implementation Details

Follow `_techspec.md` §Key flows (Status, Exit plan, Exit actions), §Integration Points, ADR-004 (amended note), ADR-010 (amended capabilities). Exit files live in `internal/worktree` per the fixed split (`exit_plan.go`, `exit_actions.go`, `forge.go`); the extension adapter implementing `ForgeProvider` lives in `internal/daemon`.

### Relevant Files

- `internal/extensionprotocol/capabilities.go` — closed provide set + service-method map (7th member lands here)
- `internal/extension/{tool_provider.go,model_source.go,watch_source.go,manager_*.go}` + `provider_conformance_matrix_integration_test.go` — consumer-registry + conformance patterns
- `extensions/dev-cycle/{embed.go,install.go,extension.json,rpc.go}` + `internal/cli/internal.go` + `internal/daemon/boot_automation_extensions.go` — bundled extension pattern
- `internal/extensionenv/bindings.go` + `internal/extension/env_bindings.go` — secret binding surface
- `internal/worktree` task-01 files — status/forge cache stores, per-repo lock, stream events
- `internal/api/core` worktree handlers (task 02) — exit route additions

### Dependent Files

- `openapi/compozy.json` + `web/src/generated/compozy-openapi.d.ts` — exit payloads (consumed by task 07)
- `internal/tools/builtin/testdata/native-tool-catalog.json` — unchanged here (verify: exit has no native tool by PRD §9 scoping)
- `docs` extension protocol pages — task 08

### Related ADRs

- [ADR-004](adrs/adr-004.md) — exit composition (amended commit-scope note)
- [ADR-010](adrs/adr-010.md) — forge surface, capabilities contract, credential model, URL-shape table
- [ADR-011](adrs/adr-011.md) — reclamation feeding cleanup suggestions

### Competitor References

- `.resources/synara/apps/server/src/git/GitManager.ts:1032-1057` (base chain), `:1265-1373` (three-layer PR idempotency), `:2598-2790` (stacked actions + phases); `apps/server/src/git/PrTemplateDetection.ts` (base-tree template walk)
- `.resources/t3code/apps/server/src/vcs/GitVcsDriverCore.ts:2766-2778` (`gh-merge-base` recorded intent)
- `.resources/paperclip/cli/src/commands/worktree.ts:2007-2150` (assess → blockers → explicit override cleanup shape)

### Web/Docs Impact

- `web/`: exit payloads/types regenerate — consumed by task 07 (S6, S14); no component work here.
- `packages/site`: extension `forge.provider` protocol docs + GitHub setup (binding + `gh`) land in task 08; generated CLI/API refs regenerate.
- QA impact: new scenarios — add content-addressed `untested` files for the exit ladder via CLI/API (commit named-untracked scope, push publish, PR idempotency, zero-credential browser tier, merged/cleanup evidence); flag only — walk in task 10.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: `forge.provider` provide surface + 3 service methods + conformance + bundled `extensions/forge/github`; no Host API changes (checked against `internal/codegen/hostapi/catalog.json` — daemon→extension only); MCP unchanged (ADR-010).
- Agent manageability: exit plan + actions + cancel on HTTP/UDS/CLI with deterministic errors; forge degradation causes machine-readable; `compozy worktree status --forge` explicit refresh.
- Config lifecycle: none — no new keys (checked: credential is an extension secret binding, not config; `[worktrees]` untouched).

## Deliverables

- Exit engine live end-to-end on API/UDS/CLI with durable ops and streamed progress
- `forge.provider` + bundled GitHub extension operational with the credential ladder
- Full parity matrix green across every worktree route
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-053–UT-063 — exit plan ladder, pauses, blocks, evidence precedence, zero-credential absence
- [x] UT-064–UT-073 — action phases, skip reasons, mid-chain failure, cancel, commit/PR semantics
- [x] UT-123–UT-127 — remote matching, credential ladder, degradation, idempotency, multi-extension winner
- [x] UT-136, UT-137 — capabilities payload completeness, browser-link ownership
- [x] UT-147 — op-id compare-and-cancel + restart survival
- [x] UT-152 — named-untracked commit scope + gitignore exclusion
- [x] IT-033 — full HTTP/UDS/CLI parity matrix (every route, every error code)
- [x] IT-035, IT-036 — forge stub conformance; gh-fallback ladder
- [x] IT-037 — exit-operation progress events in the lifecycle coverage matrix
- [x] IT-041 — remove-vs-exit-action interleaving
- [x] E2E-001 — full runtime journey through commit/push and clean removal

## Success Criteria

- Every assigned test case implemented and passing
- `make gate` green; `make codegen-check` clean; conformance matrix includes `forge.provider`
- With zero extensions: git-local exit + browser tier fully functional; with the GitHub extension + credential: PR state/create/merged live; degradation states carry causes
- No forge credential observable outside the extension process in any log, payload, event, or stream
