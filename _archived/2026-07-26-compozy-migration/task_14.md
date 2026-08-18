---
status: pending
title: "[DEFERRED] Config migrator — translate-and-drop with report, first-boot legacy state"
type: backend
complexity: high
---

# Task 14: [DEFERRED] Config migrator — translate-and-drop with report, first-boot legacy state

## Overview

**DEFERRED — do not start until product explicitly unblocks this slice.** Implements the live `compozy migrate config` verb that translates legacy configuration and drops deferred keys with an explicit report, plus the first-boot legacy-state warning and the in-place upgrade journey. Task 09 already ships `[loops.inputs.*]`, the deep link, and both migration-guide surfaces with the migrator marked deferred; this task turns the documented translate/drop matrix into executable behavior and updates the guide's migrator section from deferred to live.

<critical>
- ALWAYS READ the migration brief, TechSpec, ADRs, and test contract (`_brief.md`, `_techspec.md`, `adrs/`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS: parse legacy state only for explicit migration/probing, never as a runtime fallback; do not infer a workspace from CWD, retain inert legacy keys, or silently ignore nested deferred leaves
- DEFERRED GATE: leave `status: pending` and the `[DEFERRED]` title prefix until product unblocks; do not pull this task into the MVP or beta critical path
</critical>

<requirements>
- MUST implement `compozy migrate config [--workspace <root>]` parsing the legacy Go-struct config schema (the drifted markdown reference is NOT the source). Without `--workspace`, migrate the global config only; with it, migrate exactly that explicit workspace config instead. Migrating both requires two independently reported invocations. Never discover or mutate a workspace from CWD.
- MUST implement the canonical migration matrix below from the real v0.2.15 `ProjectConfig`: scalar runtime defaults expand to worker/judge defaults for delivery and watch; `defaults.timeout` maps to `session.limits.timeout`; `defaults.auto_commit` maps to both declared loop inputs; complexity defaults and persisted type-only task rules retain canonical order. The only review key with a declared destination is `defaults.auto_commit` — every `fetch_reviews.*` and `watch_reviews.*` key is a report-only drop because v0.3.0 review is agent-authored with no external fetch/resolve (brief round 12, ADR-008). The exact provider table applies at every runtime leaf. `droid`, `devin`, and legacy-only `ultra` are explicit unsupported-value drops, never lossy aliases. `pr` never existed as a persisted legacy default and is never fabricated.
- MUST emit exactly one report entry per populated source leaf using `{key, action, targets?, reason?, guide_ref?, redacted_value?}`. `targets` is an ordered array for one-to-many translations. Every recognized leaf not listed as translated in the canonical matrix below is a distinct reported drop — no inert keys, commented blocks, category-only receipts, or silently ignored nested fields.
- MUST back up the original with a timestamped copy before rewriting in place, and MUST refuse a second run when the backup marker exists (idempotent, non-destructive — Safety Invariant 11).
- MUST use a minimal format/topology probe before strict config decode at daemon startup and in the local CLI bootstrap for `status`/`doctor`. A legacy-format active config or legacy-owned `agents/`/`extensions/` directory at a now-colliding global/workspace `.compozy` path emits typed `legacy.state_detected {paths}` plus `migrate config`/manual-relocation remediation and blocks normal daemon startup; it is never interpreted or enrolled through a compatibility loader.
- MUST keep transport availability truthful: HTTP/UDS cannot expose diagnostics while strict config prevents the daemon from booting. After migration, a running daemon reports retained orphan legacy directories without deleting them, and that post-boot warning has identical structured CLI, HTTP, and UDS status/doctor payloads.
- MUST update both migration-guide surfaces so section 3 (migrator usage) and the config translation table describe the **live** verb, backup/idempotency, and first-boot warning — replacing task 09's deferred wording — while preserving the eight-section parity contract and `make migration-guide-check`.
- MUST NOT invent destinations for deferred product features; drops stay drops, and deferred CLI/web/SDK surfaces remain documented as deferred/removed with workarounds.
</requirements>

## Subtasks

- [ ] 14.1 Implement the explicit-scope migrator verb: legacy parse, exhaustive key matrix, translation/drop report, and no-CWD behavior
- [ ] 14.2 Implement drop reporting, timestamped backup, and idempotency refusal
- [ ] 14.3 Probe before strict decode for daemon start and local CLI status/doctor; after migration, surface retained-orphan warnings through the running daemon's shared CLI/HTTP/UDS status/doctor payload
- [ ] 14.4 Update both migration-guide surfaces from deferred-migrator wording to live migrator usage, keeping eight-section parity
- [ ] 14.5 Implement all assigned unit, integration, and E2E cases

## Implementation Details

Follow ADR-006 (migrator policy, flat namespace, first-boot posture) and TechSpec §Implementation Design for the migrator. Keep the migrator implementation under the 500-line cap by splitting parse/matrix/report/backup/probe responsibilities.

Greenfield confirmed by exploration: no `migrate` verb precedent exists in the CLI. Legacy fixtures come from the legacy tree at `the CompozyOS repo`; measure stable v0.2.15 at `8f8908afd70c731b815e20282bacad05aa026827`, audit post-tag official HEAD `c202311c8430fc0d4a7442e2dc715cabfbdc68a1` separately, reject the stale v0.2.13/v0.2.11 baselines, and include the v0.2.12 parallel-task keys in the drop matrix.

Task 09 must already have shipped `[loops.inputs.*]`, `runtime_defaults`/`runtime_rules` destinations (via task 06), and both guide surfaces before this task starts.

### Canonical Migration Matrix

| Legacy source | v0.3 target |
| --- | --- |
| `defaults.ide` | The four `loops.defaults.{delivery,watch}.runtime_defaults.{worker,judge}.provider` paths after provider mapping. |
| `defaults.model` | The four `loops.defaults.{delivery,watch}.runtime_defaults.{worker,judge}.model` paths. |
| `defaults.reasoning_effort` | The four `loops.defaults.{delivery,watch}.runtime_defaults.{worker,judge}.reasoning` paths for `low|medium|high|xhigh|max`. |
| `defaults.timeout` | `session.limits.timeout`. |
| `defaults.auto_commit` | `loops.inputs.software-delivery.auto_commit` and `loops.inputs.review-and-fix.auto_commit`. |
| `defaults.by_complexity.<level>.<ide|model|reasoning_effort>` | One ordered delivery runtime rule with `match.complexity`; empty levels emit no rule. |
| `tasks.run.task_runtime_rules[]` | Ordered delivery runtime rules with `match.type`; the persisted legacy schema permits type selectors only. |

Provider values map as `codex -> codex`, `claude -> claude`,
`cursor-agent -> cursor`, `opencode -> opencode`, `pi -> pi`,
`gemini -> gemini`, `copilot -> copilot`, and `kiro -> kiro`. Report `droid`,
`devin`, and reasoning `ultra` at their owning fields as
`unsupported_destination_value`; never alias or weaken them.

Drop every other populated recognized leaf individually with a reason and
guide reference:

- `defaults.{output_format,access_mode,tail_lines,add_dirs,max_retries,retry_backoff_multiplier}` and `defaults.stall.{enabled,timeout,child_timeout,terminal_command_timeout,retries}`;
- `tasks.types`, `tasks.run.{include_completed,recursive,output_format,run_multiple_mode,run_multiple_parallel_limit,tui}`, `tasks.run.parallel.{enabled,max_concurrency}`, and `tasks.run.parallel.conflict_resolver.{enabled,ide,model,reasoning_effort,max_attempts,validation_command}`;
- `fix_reviews.{concurrent,batch_size,include_resolved,output_format,tui}`, `fetch_reviews.{provider,nitpicks}`, and `watch_reviews.{auto_push,push_remote,poll_interval,quiet_period,max_rounds,review_timeout,until_clean,push_branch}` — the entire external-review family drops because v0.3 review is agent-authored (ADR-008);
- `exec.{ide,model,output_format,reasoning_effort,access_mode,timeout,tail_lines,add_dirs,auto_commit,max_retries,retry_backoff_multiplier,verbose,tui,persist}` and `exec.stall.{enabled,timeout,child_timeout,terminal_command_timeout,retries}`;
- `runs.{default_attach_mode,keep_terminal_days,keep_max,shutdown_drain_timeout}`, `recovery.{enabled,ide,model,reasoning_effort,max_attempts}`, and `sound.{enabled,on_completed,on_failed,on_parked}`.

Preserve explicit `false` and zero values. The legacy schema has no persisted
`defaults.provider` or `defaults.pr`; fabricate neither.

### Relevant Files

- `internal/cli/` — new migrate verb with optional explicit `--workspace`, structured `-o json` report rendering
- `internal/api/core/status.go`, `internal/api/core/doctor_payload.go`, `internal/api/contract/status.go`, `internal/api/udsapi/**`, and CLI status/doctor paths — local pre-boot legacy-config diagnostic plus post-boot orphan-warning transport parity
- `internal/daemon/**` boot path — minimal legacy-format probe before strict runtime config loading
- `extensions/dev-cycle/loops/review-and-fix/loop.yaml` and `software-delivery` declared inputs — real migrator targets
- `MIGRATION_GUIDE.md` and `packages/site/content/runtime/migration/` — update deferred migrator wording to live usage while preserving parity
- `the CompozyOS repo` — legacy `ProjectConfig` Go struct and real-shaped config fixtures
- `scripts/verify-migration-guide-parity.sh` and `Makefile` — re-run after guide updates

### Dependent Files

- `packages/site/content/runtime/cli-reference/**` — regenerated CLI reference for the live migrate verb
- `skills/compozy/references/**` — config/loops docs that previously said migration was deferred
- Task 09 guide surfaces and `[loops.inputs.*]` / runtime-default destinations — prerequisites, not re-implemented here

### Related ADRs

- [ADR-006: Flat `.compozy/` Namespace and Explicit Config Migration](adrs/adr-006.md) — translate-and-drop-with-report policy, backup/idempotency, first-boot reporting without deletion
- [ADR-007: Per-Loop Input Defaults in Configuration (`[loops.inputs.<name>]`)](adrs/adr-007.md) — declared input destinations the migrator may write; runtime selection remains engine configuration

## Deliverables

- `compozy migrate config` with structured translate/drop report, timestamped backup, and idempotency refusal
- Reachable local pre-boot legacy-config diagnostic plus post-boot orphan warnings surfaced through CLI/HTTP/UDS status and doctor, never deleting state
- Both guide surfaces updated so the migrator section describes the live verb while `make migration-guide-check` remains green
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from [`_tests.md`](_tests.md), the test contract — read each ID's full definition there before writing tests.

- [ ] UT-048–UT-050 — migrator translation: the real legacy defaults plus by-complexity, type-only persisted task rules preserving order, exhaustive review-key matrix including translated `push_remote`, and per-key drops without invented `provider`/`pr` defaults
- [ ] UT-051, UT-052 — unparseable legacy TOML fails structurally with no output written; deferred-only input yields a valid-empty config with drop-only report
- [ ] UT-053–UT-055 — drop report entries per deferred key, timestamped backup with in-place replacement, second-run refusal on the backup marker
- [ ] IT-014 — run the migrator separately against a fixture global config and an explicitly selected workspace config (including v0.2.12 keys); each invocation touches only its selected target, emitted configs load clean under the real v0.3.0 loader, reruns refuse, and no-CWD behavior holds. Before migration, daemon start and local CLI status/doctor return `legacy.state_detected` while HTTP/UDS are unavailable; colliding legacy `agents/`/`extensions/` are listed and never enrolled. After migration, retained-orphan warnings match across CLI/HTTP/UDS status and doctor while all legacy files remain intact
- [ ] E2E-003 — in-place upgrade journey: legacy machine state including colliding `agents/`/`extensions/`, preflight refusal to enroll them, separate global and explicit-workspace migration invocations, first-boot orphan reporting without deletion, delivery run with translated rules, dry-run echo showing workspace-origin input defaults

### Web/Docs Impact

- `web/`: none — checked surfaces: no web routes/components change; migrator is CLI-local pre-boot file mutation plus status/doctor diagnostics.
- `packages/site`: update `content/runtime/migration/` migrator section from deferred to live; regenerate `content/runtime/cli-reference/**` for `migrate config`; keep eight-section parity with root `MIGRATION_GUIDE.md`.
- QA impact: when unblocked, add content-addressed `untested` scenarios for the migrator run and first-boot legacy-state warning; reset any scenario whose `entry_points` cite migrate/status/doctor remediation.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: no new extension surfaces — checked manifests, hooks, tools/resources, bundles, registries, bridge SDKs; the migrator is a CLI-local file operation.
- Agent manageability: `migrate config --workspace <root> -o json` emits the structured translation/drop report; local CLI status/doctor remain available for pre-boot remediation, and running-daemon CLI/HTTP/UDS status/doctor agree on orphan warnings. The migrator is CLI-only because it edits local pre-boot files.
- Config lifecycle: the migrator owns every legacy key through an exhaustive translate-or-drop report into destinations already shipped by tasks 06 and 09 (`runtime_defaults`/`runtime_rules`, `[loops.inputs.*]`, `session.limits.timeout`).

### AGH Impact Audit

- Native tools: no general status/doctor native ToolID exists or is added; agents use the existing structured CLI/HTTP/UDS fallback, with the pre-boot diagnostic necessarily local to CLI. Regenerated CLI reference may mention the migrate verb; no new `compozy__*` migrator tool is introduced unless a later ADR adds one.
- Extensibility and hooks: no new extension, hook, bundle, registry, bridge-SDK, or MCP-sidecar surface.
- Workspace data isolation: migration writes are explicitly global or selected-workspace scoped; status/doctor diagnostics must not expose another workspace's values or paths.
- Official AGH skill: update config/loops references that previously marked migration as deferred so they describe the live `migrate config --workspace` surface.

## Success Criteria

- Every assigned test case implemented and passing
- A legacy machine upgrades in place: migrate runs, the daemon boots reporting orphan directories without deleting them, and a delivery run works from the translated configuration
- The migrator refuses a second run and never destroys the original
- Both guide surfaces describe the live migrator and stay parity-clean under `make migration-guide-check`
- `make verify` green
- `[DEFERRED]` prefix removed from the title only when product unblocks and this task is actively scheduled
