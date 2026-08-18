---
status: completed
title: "`review-and-fix` loop — agent-authored review with deterministic `reviews-NNN/` artifacts"
type: backend
complexity: high
---

# Task 07: `review-and-fix` loop — agent-authored review with deterministic `reviews-NNN/` artifacts

## Overview

Delivers the internal review journey (ADR-008): the dev-cycle `reviewer` agent reviews the work for a named task and emits the round's issues as structured `ReviewIssue[]`; source-agnostic host tools write inspectable `reviews-NNN/issue_NNN.md` artifacts; a fixer node remediates by issue-file paths; a deterministic finalize step rewrites statuses; the loop repeats until a review round comes back clean. There is **no external review provider** — every CodeRabbit fetch/resolve surface, the PR watch source, the thread-resolution and push tail, and the `gh` dependency are hard-cut in this task. Serialization lives exclusively in Go — agents edit triage content inside existing files, never create or rename them.

<critical>
- ALWAYS READ the migration brief (round 12), TechSpec, ADR-003 (as amended) + ADR-008, and test contract (`_brief.md`, `_techspec.md`, `adrs/`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS: the extension receives an explicit trusted workspace/task root; do not infer it from CWD, accept an untrusted path from an agent, or bypass containment for a first-time write.
- HARD CUT: deleting the CodeRabbit surface is in scope, not optional cleanup. No flag, alias, or abstraction may keep a fetched-provider path alive.
</critical>

<requirements>
- MUST rewrite the `review-and-fix` graph as the agent-authored journey (ADR-008): review (`run-agent` reviewer) → conditional on issues → `write_artifacts` → `fix_batches`/`fix_batch` (fan-out over issue-file paths) → `collect_fixes` → `finalize_round`. A clean review round (zero issues) terminates the loop `done`; the next round's review verifies the previous round's fixes. No watch source, no judge/verify gate, no thread resolution, no push nodes.
- MUST source each round from the dev-cycle `reviewer` agent via `output_schema`: each issue requires `title`, `body`, and `severity`; `file` and `line` are optional so general findings stay expressible. The Go writer, not the agent, owns `round`, `round_created_at`, and initial `status=pending`.
- MUST define `ReviewIssue` without provider provenance: no `provider`, provider-reference, `review_hash`, source-review-ID, or source-submitted-at fields, and no provenance-based history dedupe. Rounds are append-only; filtering repeat findings is the reviewer's responsibility, not the writer's.
- MUST hard-delete the CodeRabbit surface in `extensions/dev-cycle`: `coderabbit.go`, `coderabbit_nitpicks.go`, `coderabbit_rest.go`, `coderabbit_types.go`, the `coderabbit_fetch_unresolved`/`coderabbit_resolve_threads` tool descriptors, schemas, digests, and dispatch arms, the `new_review` watch source, the `should_push`/`push_changes` tail, and every goal/stop_when/catalog/docs copy naming CodeRabbit, threads, or PR watching. Delete `git.go`/`git_push` only if `review-and-fix` was its sole consumer — verify `software-delivery` first.
- MUST delete the retired loop inputs `pr`, `include_nitpicks`, `poll_interval`, `quiet_period`, `auto_push`, and `push_remote`. `task_name` becomes required (no `pr-<N>` derivation exists); `reviewer`/`fixer` keep agent defaults; `auto_commit` remains the only execution-mode input (local-only). Round caps use the loop contract (`iteration_cap`, `no_progress`), never a new input.
- MUST write artifacts deterministically to `.compozy/tasks/<task_name>/reviews-%03d/issue_%03d.md`: stable frontmatter field order, fixed body headings, and the unreviewed triage decision marker, verified against v0.3-owned golden fixtures. Legacy v0.2.15 byte-parity is NOT a requirement (ADR-008 decision 5); no legacy provider fallback defaults (`Review comment`, `_No review comment body provided by provider._`) exist. The writer normalizes a missing `file`/`author` to `unknown` and rejects records missing schema-required fields with a structured error.
- MUST receive authenticated workspace identity only from the daemon-to-extension envelope; no tool input or agent payload may select a root (Safety Invariant 6).
- MUST compute the next round as max(existing `reviews-%03d`)+1 with exclusive directory creation, tolerating gaps (Safety Invariant 8).
- MUST contain every write inside the task directory with a create-safe containment helper: resolve the trusted existing root/ancestor, create only controlled `reviews-NNN/issue_NNN.md` names through an `os.OpenRoot`-equivalent safe API, reject traversal and symlink escapes, and preserve macOS `/private` canonicalization (Safety Invariant 7).
- MUST declare both host tools (`write_review_artifacts`, `finalize_review_round`) non-read-only with mutating risk in the manifest, with input/output schemas and regenerated digests; remove the CodeRabbit tool descriptors in the same manifest change and bump the extension version so enrollment re-runs.
- MUST implement monotonic status transitions `pending → valid|invalid → resolved`, rewritten in place and idempotent on re-run (Safety Invariant 8).
- MUST keep serialization authority in Go — agents mark triage/decision inside existing files only (Safety Invariant 9).
- MUST rewrite both agent prompts: `reviewer/AGENT.md` generates the round's `ReviewIssue[]` for the named task (it is the round's source, not a verification judge); `review_fixer/AGENT.md` and the `fix_batch` prompt/output schema drop `provider_ref` and all provider/thread/push language while keeping the file-paths batch contract (`path`, `triage`, `resolution`, `summary`).
- MUST rename the loop to `review-and-fix` as a hard cut: directory, manifest entry, catalog copy, and every reference; the old loop name and the watch semantics are both delete targets.
- MUST NOT implement a `_meta.md` reader — round metadata reconstructs from issue frontmatter.
- MUST NOT add a generic artifact-writing primitive to the loop engine (ADR-003 rejected alternative 1).
- MUST NOT create any `gh` PATH-shim fixture — the loop has no `gh` boundary; delete any shim work inherited from the superseded plan.
</requirements>

## Subtasks

- [x] 7.1 Define the provenance-free `ReviewIssue` record, writer-owned metadata/normalization, and the two tool input/output schemas
- [x] 7.2 Implement the deterministic writer with v0.3-owned golden fixtures
- [x] 7.3 Implement round discovery and exclusive round-directory creation (append-only rounds)
- [x] 7.4 Propagate trusted workspace context and implement create-safe containment for every write path, including nonexistent first round, symlink, traversal, and macOS canonicalization cases
- [x] 7.5 Implement the finalize tool with monotonic, idempotent status rewriting and counts
- [x] 7.6 Register both tool descriptors (schemas, digests, mutating risk), delete the CodeRabbit descriptors, and rewire dispatch
- [x] 7.7 Consume task 06's published frontmatter payload; rewrite `loops/review-and-fix/loop.yaml` — agent-authored graph, required `task_name`, surviving inputs, clean-round termination contract, CodeRabbit-free catalog copy
- [x] 7.8 Rewrite `reviewer`/`review_fixer` AGENT.md prompts and the `fix_batch` prompt/output schema; update the affected embed-test contract assertions
- [x] 7.9 Delete `coderabbit*.go`, their types/tests/testdata, and every remaining `gh`/PR/thread reference in the extension
- [x] 7.10 Implement all assigned unit, integration, and E2E cases

## Implementation Details

Follow ADR-008 plus ADR-003 (as amended), TechSpec §Implementation Design > Data Models (review artifact contract), §Safety Invariants 6-9, and §Development Sequencing step 3. New Go files stay under the 500-line cap — split writer, finalizer, and schema/dispatch concerns rather than growing existing extension files.

Prerequisite: task 06 owns and publishes the extended task-frontmatter/import payload. Do not edit that parser in this task; consume the completed descriptor contract.

**Worktree state to inherit:** a partial implementation of the superseded CodeRabbit-first plan exists uncommitted — `review_artifact_{types,schemas,writer,paths,history,finalize}.go` and `review_artifacts_test.go` are new files, `loops/reviews-watch/loop.yaml` is deleted, `loops/review-and-fix/loop.yaml` exists with the CodeRabbit-shaped graph (`new_review` → `fetch_issues` → … → `resolve_threads` → `should_push` → `push_changes`), the superseded plan also created `internal/testutil/e2e/gh_shim.go` and `internal/daemon/daemon_review_and_fix_e2e_integration_test.go` (both untracked, both delete targets), and `coderabbit*.go`/`schemas.go`/`runtime.go`/`extension.json`/`embed_test.go`/agent prompts carry in-progress edits. Keep and adapt the writer/finalizer/containment work (it matches this task); relocate `ReviewIssue` out of `coderabbit_types.go` before deleting that file; strip provider provenance from `review_artifact_types.go` and delete `review_artifact_history.go` (provenance dedupe); rewrite the loop YAML and agent prompts; delete the CodeRabbit files outright. The generic watch-source engine (`internal/loop/watch/`, `internal/loop/coordinator_watch*.go`, `internal/extension/watch_source.go`) stays untouched — it is a supported product surface for user-authored loops and is OUT OF SCOPE for this task; the only permitted change there is swapping a `coderabbit_pr_review` fixture string in engine tests for a synthetic kind (behavior-preserving fixture rename, no engine code changes).

### Relevant Files

- `extensions/dev-cycle/loops/review-and-fix/loop.yaml` — rewrite: graph, inputs, contract (goal/definition_of_done/stop_when on clean review round), catalog copy
- `extensions/dev-cycle/review_artifact_types.go`, `review_artifact_schemas.go`, `review_artifact_writer.go`, `review_artifact_paths.go`, `review_artifact_finalize.go`, `review_artifacts_test.go` — keep/adapt (provenance removal, schema-required rejection, v0.3 goldens)
- `extensions/dev-cycle/review_artifact_history.go` — delete if provenance-dedupe-only
- `extensions/dev-cycle/coderabbit.go`, `coderabbit_nitpicks.go`, `coderabbit_rest.go`, `coderabbit_types.go` — delete targets, with their tests and testdata; `ReviewIssue` relocates out of `coderabbit_types.go` first, and the shared helpers hosted in `coderabbit.go` (`normalizePR`, `splitRepo`, `requestedWorkspaceRoot`) die with their last consumers
- `extensions/dev-cycle/schemas.go`, `runtime.go`, `rpc.go`, `extension.json`, `embed_test.go`, `provider_test.go` — descriptor/dispatch/watch-kind-advertising/manifest/pinned-contract updates; `capabilities.provides` drops `loop.watch_source`
- `extensions/dev-cycle/command.go` — whole-file delete once `coderabbit.go` and `git.go` are gone (zero remaining consumers; verify)
- `extensions/dev-cycle/agents/reviewer/AGENT.md`, `agents/review_fixer/AGENT.md` — prompt rewrites per ADR-008
- `extensions/dev-cycle/git.go` — delete; `review-and-fix` was its sole consumer (re-verify `software-delivery` references first)
- `extensions/dev-cycle/testdata/review_artifacts/{coderabbit,defaults}.md` — replace both goldens with v0.3-shape fixtures (they carry provider provenance and the retired fallback defaults)
- `internal/daemon/native_extension_tool_provider.go` — remove the CodeRabbit/`git_push` tool-ID consts and switch arms; keep trusted-workspace attach for the surviving tools
- `internal/testutil/e2e/gh_shim.go` and `internal/daemon/daemon_review_and_fix_e2e_integration_test.go` — delete (superseded-plan artifacts); the shim self-test in `internal/testutil/e2e/runtime_harness_helpers_test.go` goes with them
- `internal/daemon/**`, `internal/extension/**` — trusted workspace identity propagation into mutating extension calls; two-workspace isolation coverage; `loop_tool_schema_source_test.go` and `native_extension_tool_provider_test.go` fixture updates
- `internal/testutil/e2e/**` and `internal/e2elane/lanes.go` — E2E-002 daemon harness and lane registration (fresh acpmock-based journey)

### Dependent Files

- `internal/extension/**` manifest checksum and tool-registration paths — version bump re-enrolls the extension
- `web/src/systems/loops/**` — loop catalog copy shows the renamed loop (display only); CodeRabbit-flavored fixtures/copy in `mocks/fixtures.ts`, `components/stories/loop-run-page-fixture-world.ts`, and `components/stories/loop-editor.stories.tsx` are rewrite targets; do not add a `schedule` start kind (two web tests assert its absence)
- `packages/site/content/runtime/core/loops/{index,catalog,extensions}.mdx` — CodeRabbit/watch copy rewrites
- `skills/compozy/references/loops.md` — official-skill guidance for the agent-authored loop (replaces the gh-auth/PR-watch paragraph)

### Related ADRs

- [ADR-008: Agent-Authored Review — `review-and-fix` Without External Providers](adrs/adr-008.md) — the loop's source, graph, inputs, provenance removal, and delete targets
- [ADR-003: `reviews-NNN/` Artifacts — Source-Agnostic Dev-Cycle Writer, File-Based Fixer](adrs/adr-003.md) (as amended) — writer authority, containment, rounds, finalization, no-`_meta.md`

## Deliverables

- Two host tools (write artifacts, finalize round) with schemas, digests, mutating risk, and dispatch wiring; zero CodeRabbit tools remaining
- Deterministic `reviews-NNN/issue_NNN.md` artifacts verified against v0.3-owned goldens
- Atomic round numbering and path-contained writes
- `review-and-fix` loop delivering review → write → fix(paths) → finalize in one run, terminating on a clean review round
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from [`_tests.md`](_tests.md), the test contract — read each ID's full definition there before writing tests.

- [x] UT-033–UT-036 — writer byte-shape goldens (v0.3-owned, provenance-free frontmatter), required-`task_name` rejection, `unknown` normalization plus schema-required rejection, empty-issues no-op
- [x] UT-037, UT-038, UT-040 — round discovery with gaps, concurrent exclusive creation, append-only rounds with no provenance dedupe
- [x] UT-039 — trusted-root path containment rejection for traversal and escaping symlinks
- [x] UT-041–UT-044 — finalize: in-place rewrite preserving other bytes, pending untouched and counted, idempotent re-run, deterministic not-found error
- [x] IT-009 — reviewer agent (via `acpmock`) emits issues → writer produces byte-golden files and a fan-out payload of paths; two-workspace isolation
- [x] IT-010 — stub fixer marks statuses, finalize resolves both, counts correct, round meta reconstructed from frontmatter only
- [x] IT-011 — schema-invalid reviewer output fails the node with a structured error; no partial round
- [x] IT-012 — full `review-and-fix` graph via `acpmock`: issues → artifacts → fix → finalize → clean second round → `done`, with no watch/resolve/push nodes, including two-workspace isolation
- [x] E2E-002 — review remediation journey end-to-end with byte-checked artifacts and clean-round termination

### Web/Docs Impact

- `web/`: loop catalog and run views display the renamed loop with its new description; no new controls, payload fields, or hooks. Checked surfaces: `web/src/systems/loops/**`; reason: the rename and copy surface through existing loop-name display paths.
- `packages/site`: authored docs referencing `reviews watch` or CodeRabbit-driven review update to the agent-authored `review-and-fix`; the command mapping is owned by the migration guide (task 09); `skills/compozy/references/loops.md` documents the agent-authored loop.
- QA impact: new scenarios — add content-addressed `untested` files for the agent-authored `review-and-fix` run, on-disk artifact inspection, and round finalization; reset any existing scenario whose `entry_points` cite `reviews-watch` or CodeRabbit.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: `extension.json` gains the two writer/finalizer tool descriptors (mutating, non-read-only) with new schemas and digests and **loses** the two CodeRabbit tool descriptors; the loop resource is renamed; the extension version bump plus checksum change re-enrolls the bundled install at boot. No new hook events — the review lifecycle is expressed as loop nodes, not hooks (verified).
- Agent manageability: the two tools are agent-callable through the extension tool surface with typed input/output; `loop run --workspace <ref> --name review-and-fix --input task_name=<slug>` works over CLI, HTTP, and UDS identically; finalize returns structured counts `{resolved, invalid, pending}`; the not-found error is deterministic and names task and round.
- Config lifecycle: no new `config.toml` keys in this task — the loop's declared inputs (`task_name`, `reviewer`, `fixer`, `auto_commit`) keep their definition defaults. Workspace-level defaults for those inputs arrive in task 09 via `[loops.inputs.review-and-fix]`.

### Compozy Impact Audit

- Native tools: no native `compozy__*` IDs, toolsets, descriptors, schema digests, capability gates, or CLI/API fallbacks change; checked `compozy__loop_run` and `compozy__loop_status`, whose generic loop contracts already carry the renamed enrolled loop.
- Extensibility and hooks: dev-cycle gains the two mutating `ext__dev_cycle__*` writer/finalizer descriptors and deletes the two CodeRabbit descriptors — schemas, digests, risks, trusted workspace authorization, implementations, and the renamed agent-authored loop; no generic loop-engine writer, hooks, bundles, bridge SDKs, or MCP sidecars are added.
- Workspace data isolation: artifact reads/writes are workspace-scoped through the daemon-to-extension invocation; prove a caller cannot read or create another workspace's task artifacts.
- Official Compozy skill: update the review-loop guidance so review agents emit `ReviewIssue[]`, fixer agents edit only existing triage, finalization alone transitions to resolved, and no CodeRabbit/PR/`gh` step is documented.

## Success Criteria

- Every assigned test case implemented and passing
- Generated issue files are byte-identical to the v0.3-owned goldens, with deterministic field order and no provider provenance keys
- Concurrent writers never share a round directory and never interleave files
- A full `review-and-fix` run remediates and terminates `done` on a clean review round with zero external processes (`gh`, CodeRabbit, network) involved
- Zero `reviews-watch`, `coderabbit`, or PR-watch references remain in the extension; `make verify` green
