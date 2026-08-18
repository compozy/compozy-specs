---
status: completed
title: Schema and worktree domain core
type: backend
complexity: critical
---

# Task 1: Schema and worktree domain core

## Overview

Delivers the physical model and the entire `internal/worktree` domain: the Goose migration (worktree tables, composite binding FKs, run snapshot columns, profile policy columns with the real coupled CHECK), the hardened capturing git runner with the per-repository lock, and every lifecycle flow — phased creation with cancel/rollback, bootstrap, strict adoption, discovery/status, fenced two-step removal with branch reclamation, reconcile-never-cascade, boot recovery, dismiss — plus the `worktree` hook family, canonical events, and the `[worktrees]` config section. Everything else in the program builds on this slice.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST append the next gap-free Goose migration to the global stream (head was `00059_*`; verify the current head at implementation time) creating `worktrees` (with `git_dir`, `pending_phase`, `run_namespace`, `UNIQUE(workspace_id, name)`, `UNIQUE(workspace_id, id)`, live-path partial unique index), `worktree_status`, `worktree_forge_status`, `worktree_exit_ops` (single-active partial unique index), nullable `worktree_id` binding columns on `sessions`/`task_runs` with **composite FKs** `(workspace_id, worktree_id) → worktrees(workspace_id, id)` (table rebuild), `task_runs.resolved_worktree_mode/_ref` snapshot columns, and `task_execution_profiles.worktree_mode/_ref` with the real coupled CHECK — declarative fragments + `atlas.sum` + sqlc via `make codegen` (activate `eng-schema-migration`).
2. MUST create the `internal/worktree` package with the file split fixed in the TechSpec (Architectural Boundaries: 19 named files, hard cap 500 lines each) and no imports of `session`/`task`/`loop`/`workspace`/`daemon`/`api`/`cli`.
3. MUST implement the capturing git runner per ADR-008: `execabs.LookPath`, version floor 2.37, per-call timeouts, `Stdin=nil`, `GIT_TERMINAL_PROMPT=0`/`GCM_INTERACTIVE=never`, strip only `GIT_DIR`/`GIT_WORK_TREE`/`GIT_INDEX_FILE`, preserve user config; capability gating degrades the whole surface to `worktree_git_unavailable`/`worktree_git_version_unsupported` diagnostics.
4. MUST serialize every git mutation per repository under the canonical-common-dir lock (bounded fail-fast waiters → `worktree_operation_in_progress`); discovery and status reads never take it and never mutate git state (no implicit fetch/prune).
5. MUST implement creation as the durable phased state machine (`pending_phase`: branch → checkout → copy → setup) with `CancelCreate` (pending-only, `worktree_not_pending` otherwise), full reverse unwind on failure, `created_head`/`base_ref`/`run_namespace`/`git_dir` recording, `git config branch.<b>.gh-merge-base <base>` for minted branches, and the `worktree.pre_create` sync gate (`worktree_denied_by_hook`).
6. MUST implement the bootstrap contract: ignored-only copy list (`git ls-files --others --ignored --exclude-standard -z -- <patterns>`, no-overwrite, nesting guard), setup command via the operator's shell with the minimal allowlisted env, size-capped redaction-scrubbed output, process-group kill+wait on `setup_timeout`; failure ⇒ `setup_state=failed`, usable-but-flagged.
7. MUST implement adoption per the strict identity contract (TechSpec Adopt flow): exact porcelain path match + bidirectional admin-gitdir linkage, classified refusals (`adoption_main_checkout`/`adoption_foreign_repository`/`adoption_unreadable`), no bootstrap, no mutation of refused directories, idempotent re-adopt.
8. MUST implement removal with the `ready → removing` CAS fence, DB-fence-then-repo-lock order, under-lock authoritative re-evaluation, two-step refusals with risk inventory (`worktree_dirty_requires_force`/`worktree_unpushed_requires_force`, remote-exists downgrade), `worktree.pre_remove` sync gate, gitdir-verified leftover recovery, compare-and-delete branch reclamation (`update-ref -d` against `created_head`, only `created_branch=1` + recorded namespace), and `removed` tombstones.
9. MUST implement reconcile-never-cascade (`missing` flips only; restore requires `git_dir` fingerprint match), `RecoverCreations` phase-driven boot recovery (never infers readiness from disk), and `Dismiss` tombstoning — no path may ever route worktrees through the workspace prune/`Unregister` cascade.
10. MUST register the `worktree` hook family (5 events: `pre_create`/`pre_remove` syncEligible, `created`/`adopted`/`removed` observe-only) in both hook registry spots, the canonical event entries (exhaustive list in TechSpec Monitoring), payloads + introspection descriptors + dispatch at the mutating call sites, and `worktree_id` in correlation keys.
11. MUST add `WorktreesConfig` (`root`, `run_branch_namespace`, `copy_list`, `setup_command`, `setup_timeout`, `discovery_cache_ttl`) with defaults, overlay merge, validation naming the offending key, lifecycle-matrix `Live` entries, and example configs.
12. MUST predicate every Service read/mutation and store query on `(workspace_id, worktree_id)` — a mismatch returns `worktree_not_found` with no existence leak; no raw claim token or secret may ever enter the worktree domain, diagnostics, events, or persisted strings (shared redaction boundary).
13. MUST implement naming per ADR-011: shared sanitize function, adjective-noun wordlists with `-N` collision suffix, per-run `<ns><task-slug>-<run-suffix>` composition, placement `<root>/<workspace-name>/<name>`, and creation-path approved-roots containment re-validated before every mutation.
</requirements>

## Subtasks

- [x] 1.1 Declarative schema fragments + Goose migration + `atlas.sum` + sqlc regen (`make codegen`), with fresh/reopen/ahead/integrity/equivalence coverage extended and direct-rejection constraint cases
- [x] 1.2 Package scaffold per the fixed file split: entity/states/sentinel errors/Store interface, service struct + functional options, boundaries rules in `magefiles/boundaries.go`
- [x] 1.3 Git runner + capability gate + porcelain parsers (worktree list `-z`, status v2, numstat) with table-driven fixtures
- [x] 1.4 Naming: sanitize, wordlists, namespace composition, placement derivation, approved-roots checks
- [x] 1.5 Per-repo mutation lock keyed on canonical common dir with bounded fail-fast waiters
- [x] 1.6 Phased creation + bootstrap (copy list, setup command, env allowlist, timeouts, process-group cleanup) + `CancelCreate` + unwind
- [x] 1.7 Adoption with strict identity validation and `git_dir` fingerprint recording
- [x] 1.8 Discovery/list merge (registry + cached git scan, TTL, refresh bypass, prunable-stale classification) + status reads with NULL-unknown semantics and `read_error`
- [x] 1.9 Removal: `removing` fence, safety evaluation, two-step refusals, force path, leftover recovery, branch reclamation, tombstones
- [x] 1.10 Reconcile (missing/restore), `RecoverCreations`, `Dismiss`
- [x] 1.11 Hook family + canonical events + payloads + call-site dispatch + coverage-matrix rows + redaction boundary
- [x] 1.12 `[worktrees]` config lifecycle (struct, defaults, overlay, validation, matrix, examples)

## Implementation Details

Authoritative design: `_techspec.md` §Implementation Design (Core Interfaces, Data Models, Key flows), §Safety Invariants 1-19, §Config Lifecycle. Daemon wiring for the service happens here only to the extent of construction (`internal/daemon` boot); consumer seams land in tasks 02-05.

### Relevant Files

- `internal/store/globaldb/schema/definitions/` + `schema/migrations/` (head `00059_*`, `atlas.sum`) — migration home; `00_core.sql:7-18` workspaces table shape; `20_sessions.sql:104`, `72_task_runs.sql:77` FK precedents; `71_task_profiles.sql:19-39` coupled-CHECK precedent
- `internal/workspace/registration_id.go:10-29` — `generateID("wt")` pattern; `identity.go`, `resolver_reconcile.go:9-49` (prune path to quarantine from)
- `internal/registry/gitsrc/git_runtime.go:12,20-33,92-122` + `client.go:33-43,246-256` — runner posture to copy (floor 2.37, env hardening, injectable seams; add output capture, keep user config)
- `internal/hooks/events.go:8-56,61-154,162-383` — family + Validate + definitions; `payloads_sandbox.go`, `dispatch_tasks.go`, `introspection_descriptors_*.go` patterns
- `internal/events/registry.go:21-93` + `registry_workspace_access.go:4-11` — canonical event entry idiom; add `registry_worktree.go`
- `internal/config/config_extensions_sandbox.go:75-108`, `merge_overlay.go:3-60`, `defaults.go`, `config_validation.go`, `lifecycle/lifecycle.go:14-93`, `config_load.go:38-44` (workspace overlay) — config lifecycle homes
- `internal/fileutil` — `AtomicWriteFile`, `PathWithinRoot`, canonicalization helpers for containment/identity comparisons
- `internal/procutil` — process-group signaling for setup-command cleanup

### Dependent Files

- `internal/daemon/boot_*.go` — service construction + `RecoverCreations` at boot (consumer wiring expands in tasks 02-05)
- `magefiles/boundaries.go` — new-package import rules (same commit as the package)
- `internal/store/globaldb/sqlcgen/` — regenerated queries
- `internal/tools/builtin/testdata/native-tool-catalog.json` — untouched here (task 02 owns it); listed to flag the shared digest surface

### Related ADRs

- [ADR-007](adrs/adr-007.md) — storage model, tombstones, prune quarantine
- [ADR-008](adrs/adr-008.md) — git runner, per-repo lock, capability gating
- [ADR-011](adrs/adr-011.md) — namespace, generated names, compare-and-delete reclamation
- [ADR-002](adrs/adr-002.md) / [ADR-006](adrs/adr-006.md) — adoption semantics, lifecycle safety (amended scope for setup visibility)

### Competitor References

- `.resources/synara/apps/server/src/git/Layers/GitCore.ts:781-834` (per-repo lock), `:2631-2736` (phased create + rollback), `:2822-2894` (compare-and-delete), `:2220-2402` (copy list + transfer)
- `.resources/herdr/src/worktree.rs:404-499` (porcelain parse), `:158-226,328-375` (dirty classification + leftover recovery), `src/workspace/git/discovery.rs:42-158` (common-dir identity without subprocess)
- `.resources/t3code/apps/server/src/vcs/GitVcsDriverCore.ts:2482-2527` (`--git-dir <commonDir> worktree list --porcelain -z`), `:2751-2786` (create + gh-merge-base), `apps/web/src/worktreeCleanup.ts:11-33` + `apps/web/src/hooks/useThreadActions.ts:272-469` (the force-remove negative example)

### Web/Docs Impact

- `web/`: none in this task — backend domain only; web consumption lands in tasks 06-07 (checked surfaces: no contract/OpenAPI change ships here; routes arrive in task 02).
- `packages/site`: none in this task — docs land in task 08 (config keys documented there).
- QA impact: none — no user-visible behavior change until surfaces ship (tasks 02+ flag scenarios).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: `worktree` hook family (5 events, sync flags per TechSpec) + canonical events + introspection descriptors; no extension manifest/tool/resource changes here (forge surface is task 05; checked: `internal/extension*` untouched).
- Agent manageability: none yet by design — the agent-operable surface (CLI/HTTP/UDS/tools) is task 02; this task ships the domain those surfaces call.
- Config lifecycle: `[worktrees]` section complete (struct, defaults, overlay incl. per-workspace `.compozy/config.toml`, validation, lifecycle matrix `Live`, example configs, tests UT-111..114); docs page lands in task 08.

## Deliverables

- Migration + declarative schema + sqlc landing `make codegen-check` clean
- `internal/worktree` package implementing every flow above within the fixed file split
- Hook family + canonical events registered with coverage-matrix rows
- `[worktrees]` config lifecycle complete
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-001–UT-006 — naming, slugging, namespace eligibility
- [x] UT-007–UT-012 — capability gate + git runner env/timeout contract
- [x] UT-013–UT-020 — porcelain parsers incl. error paths
- [x] UT-021–UT-036 — creation sequencing, refusals, rollback, bootstrap
- [x] UT-037–UT-048 — adoption validation, discovery merge, cache
- [x] UT-049–UT-052 — status assembly, unknown semantics
- [x] UT-074–UT-087, UT-135 — removal, reclamation, reconcile, recovery, dismiss
- [x] UT-111–UT-114 — config defaults/overlay/validation/lifecycle
- [x] UT-116–UT-118, UT-138–UT-140 — hook family, registry, deny/fail-open
- [x] UT-143–UT-146, UT-148–UT-150 — workspace predication, phases, cancel, removing fence, strict adoption, approved roots, secret non-egress
- [x] IT-001–IT-025 — real-git lifecycle, bootstrap, races, locks, migration suites
- [x] IT-037 — core-domain lifecycle events (exit progress completes in task 05)
- [x] IT-041 — remove-vs-same-branch-create + external-commit TOCTOU cases (session/exit interleavings complete in tasks 03/05)

## Success Criteria

- Every assigned test case implemented and passing
- `make gate` green (scoped Go lanes); `make codegen-check` clean; migration suites green under `make test-integration`
- Safety invariants 1-19 of the TechSpec each traceable to at least one passing test in this task's set
- Zero imports from `internal/worktree` into forbidden packages (`mage Boundaries` green)
