---
schema_version: "compozy.tasks/v2"
workflow: compozy-migration
graph:
  nodes:
    - id: task_01
      file: task_01.md
    - id: task_02
      file: task_02.md
    - id: task_03
      file: task_03.md
    - id: task_04
      file: task_04.md
    - id: task_05
      file: task_05.md
    - id: task_06
      file: task_06.md
    - id: task_07
      file: task_07.md
    - id: task_08
      file: task_08.md
    - id: task_09
      file: task_09.md
    - id: task_10
      file: task_10.md
    - id: task_11
      file: task_11.md
    - id: task_12
      file: task_12.md
    - id: task_13
      file: task_13.md
    - id: task_14
      file: task_14.md
  edges:
    - from: task_01
      to: task_02
    - from: task_02
      to: task_03
    - from: task_03
      to: task_04
    - from: task_04
      to: task_05
    - from: task_05
      to: task_06
    - from: task_06
      to: task_07
    - from: task_06
      to: task_09
    - from: task_07
      to: task_08
    - from: task_07
      to: task_09
    - from: task_08
      to: task_10
    - from: task_09
      to: task_10
    - from: task_09
      to: task_14
    - from: task_10
      to: task_11
    - from: task_11
      to: task_12
    - from: task_12
      to: task_13
---

# CompozyOS Migration Task List

Execution frame (brief rounds 8 + 11): the whole tree is copied to branch `v0.3` of `github.com/compozy/compozy` **before** task 01 starts; `main` stays on v0.2.x throughout implementation. Tasks 01-05 are the zero-legacy rebrand hard cut (one class per task, each with its own grep gate). Tasks 06-09 deliver active P0 parity. Tasks 10-11 prepare the **single v0.3 beta cut entirely in-branch** — release mechanics, front door, and launch content ship in the same tree; no task adds an artifact another task removes. Tasks 12-13 validate the exact pre-publish revision. Task 14 is a deferred post-MVP branch for the live config migrator and does not block tasks 10-13. Only after Task 13 is green may Pedro execute Task 10's single-cut runbook (`legacy/v0.2` branch, squash merge of `v0.3` into `main`, beta publish, `compozy.com` pointing, `compozy/agh` archival) with its mandatory live registry/install/signature checks. Every implementation task ends with `make verify` green.

## MVP Boundary

- **MVP — tasks 01-09.** Phase 1 (rebrand: identity, namespaces, wire protocol, npm/web/site, governance content) plus Phase 2 (P0 parity: per-task runtime selection, `review-and-fix` with `reviews-NNN/` artifacts, bundled skills, per-loop input defaults, migration guide on both surfaces, web deep link). The live `compozy migrate config` verb is **not** in MVP.
- **Release cut — tasks 10-11 (brief round-11: single cut, in-branch).** Task 10 owns release identity, the pinned `github.com/compozy/releasepr@v0.0.24` release-plan contract, the single-contract installer, the beta front-door README with the `legacy/v0.2` deprecation pointer, and the single-cut runbook; every shipped install surface points at the beta channel and none offers Homebrew until the post-beta 0.3.0 stable bump. Task 11 ships the launch content live on the branch: locked hero, six-slot landing, drawn wordmark, launch post, and "The OS Release" notes. The planner validates explicit ref/version/channel inputs and emits authoritative GitHub/npm/Homebrew policy, but the *external* execution — annotated tag creation, `legacy/v0.2` branch, squash merge into `main`, beta publish, `compozy.com` pointing, `agh.network` retirement, and `compozy/agh` archival — remains in the workflow and the gated runbook, executed only after Task 13. The post-beta stable release (npm `latest`, Homebrew bump via `channel=stable`) is normal release work outside this spec.
- **QA — tasks 12-13.** `qa-report` plans the cycle over the living `docs/qa/` tree; `qa-execution` runs it against local/pre-publish artifacts and a fresh release-PR dry run. Task 13 green unlocks the Task 10 single-cut runbook; it does not claim that a beta already exists. Migrator/first-boot upgrade journeys are out of this cycle until task 14 is unblocked.
- **Deferred post-MVP — task 14.** Live config migrator (`compozy migrate config`), first-boot legacy-state probe/orphan warnings, and E2E-003 in-place upgrade. Documented in the task 09 guide as deferred; does not block beta/stable staging.
- **Post-MVP (explicitly out of scope):** TUI cockpit/attach, `reviews fetch/list/show/fix` verbs, loop-run wizard, `--multiple`/`--parallel` worktree isolation, `--parallel-tasks` wave execution, recovery agent, the existing legacy review-provider abstraction removed with no external review fetch/resolve in v0.3 at all — review is agent-authored (brief round 12, ADR-008), cost-aware budgets, agent-per-task rules, and external-CLI skill install helper. Each is documented with status and workaround in the migration guide (task 09).
- **Removed legacy ecosystem (document, do not rebuild):** the public Go facade, legacy Go/TypeScript extension SDKs, `@compozy/create-extension`, `compozy ext`, legacy web route families, and first-party extensions not in the approved nine-skill/dev-cycle bundle receive explicit successor-or-no-successor rows. The API-incompatible v0.3 SDK/scaffolder workspaces are private and are not published over legacy npm identities.

## Tasks

| # | Title | Status | Complexity | Dependencies |
| --- | --- | --- | --- | --- |
| 01 | Rebrand core identity — home dir, database, binary, module path | completed | high | - |
| 02 | Rebrand namespaces — environment variables and native tool IDs | pending | high | task_01 |
| 03 | Rebrand wire protocol and protocol documents | pending | high | task_02 |
| 04 | Rebrand workspace packages, web, documentation site, and official runtime skill | pending | high | task_03 |
| 05 | Rebrand authored governance content and project memory | pending | medium | task_04 |
| 06 | Per-task runtime selection — engine, binder, contract, CLI, web | pending | high | task_05 |
| 07 | `review-and-fix` loop — agent-authored review with deterministic `reviews-NNN/` artifacts | pending | high | task_06 |
| 08 | Bundled skills inside the dev-cycle extension | pending | medium | task_07 |
| 09 | Per-loop input defaults, migration guide, deep link | pending | high | task_06, task_07 |
| 10 | Prepare the v0.3 beta cut — release identity, channel mechanics, single-cut runbook | pending | high | task_08, task_09 |
| 11 | Launch content in-branch — landing, wordmark, launch post, release notes | pending | medium | task_10 |
| 12 | QA Plan and Session Charters | pending | high | task_11 |
| 13 | Real-User QA Execution | pending | critical | task_12 |
| 14 | [DEFERRED] Config migrator — translate-and-drop with report, first-boot legacy state | pending | high | task_09 |

## Test Contract Assignment

Every ID in [`_tests.md`](_tests.md) is assigned to exactly one task (66 UT + 18 IT + 4 E2E). Deferred task 14 owns the migrator cases; they are not MVP gates.

| Task | Unit | Integration | E2E |
| --- | --- | --- | --- |
| task_01 | — (existing suites co-ship; gates own the invariants) | — | — |
| task_02 | UT-057, UT-058 | IT-016 | — |
| task_03 | UT-059 | — | — |
| task_04 | — (the complete currently discovered pinned site-spec suite co-ships) | — | — |
| task_05 | — (grep gates) | — | — |
| task_06 | UT-001–UT-032, UT-060–UT-062 | IT-001–IT-008 | E2E-004 |
| task_07 | UT-033–UT-044 | IT-009–IT-012 | E2E-002 |
| task_08 | UT-045–UT-047 | IT-013 | — |
| task_09 | UT-056, UT-063–UT-066 | IT-015, IT-018 | E2E-001 |
| task_10 | — | IT-017 | — |
| task_11 | — (site specs co-ship) | — | — |
| task_14 | UT-048–UT-055 | IT-014 | E2E-003 |

## Verified Seam Divergences

Codebase exploration (2026-07-24) found the TechSpec cites several seams from memory. The design is unaffected; task bodies carry the real paths:

1. `ResolveSessionAgentWithRuntime` lives at `internal/config/provider_resolve.go:219`; `internal/config/provider.go` also exists but is not the resolver owner.
2. `CanonicalProviderModelName` **does not exist anywhere**; `CanonicalProviderName` is at `internal/config/provider_builtin.go:234` (not in `modelcatalog`). Model validation today is `modelcatalog.HasAuthoritativeProviderCatalog` (cursor-only allowlist) + `internal/session/runtime_model_preflight.go`. The `RuntimeCatalog` implementation is net-new (task 06).
3. Database names live in `internal/store/store_paths.go:7,9` — `internal/store/store.go` is a package-doc stub.
4. Verify-lock dir is defined in `magefiles/verifylock.go:17-19`, not a Makefile literal.
5. No `agh.network.v0` NATS subject constant exists. Real hard-cut surfaces include `ProtocolV0` (`internal/network/envelope.go:12`), `RuntimePeerID` plus its derived session ID (`manager.go:19-20`), the `agh.*` envelope extension keys, `agh.loop/v1`, the soul/heartbeat digest and persistence prefixes, `agh-network/direct-room/v1\0`, and the trust profile `agh-network.trust.ed25519-jcs/v1`.
6. `packages/site/content/runtime/migration/` absent and **no redirect infrastructure exists** (no `redirects()` in `next.config.mjs`, no root `vercel.json`).
7. `skills/agh/` contains the official `SKILL.md` plus its reference inventory; discover the current set at execution time rather than pinning a fragile count.
8. `packages/site/lib/__tests__/` is the canonical pinned-spec inventory; discover and run the complete current suite rather than relying on a historical count.
9. `loop run` has no model flags and the web run view renders no runtime — those two display/input surfaces are greenfield. Existing `model_defaults` plumbing and judge-model fields are delete targets, not greenfield.
10. No `migrate` verb precedent in `internal/cli/`; no `gh` PATH shim in tests.
11. dev-cycle loop node ids include `fetch_issues`, `new_review`, `fix_batches`/`fix_batch`, `collect_fixes`, `verify`, `resolve_threads`, `should_push`, and `push_changes`; `agents/reviewer/AGENT.md` carries zero skill references (unlike its two siblings). *(Dated pre-ADR-008 observation: after brief round 12, every node in that list except `fix_batches`/`fix_batch`/`collect_fixes` is a delete target — the agent-authored graph is review → write_artifacts → fix → finalize.)*
12. `make codegen` is a graph, not a four-artifact command: it owns Atlas/sqlc, Daytona sidecars, `compozy-codegen all` (`openapi`, `sdk-contracts`, `loop-enums`, `lifecycle-matrix`, `native-tool-catalog` after task 01), generated TypeScript OpenAPI declarations, and DESIGN sync. Tasks name affected families but always use `make codegen-check` for drift.
13. The legacy reference is stable tag `v0.2.15` at commit `8f8908afd70c731b815e20282bacad05aa026827`, with post-tag official HEAD `c202311c8430fc0d4a7442e2dc715cabfbdc68a1` audited separately; `v0.2.13` is not the migration baseline.
14. Loop HTTP/UDS routes are workspace-scoped (`/api/workspaces/{workspace_id}/loops/{name}/run|validate` and `/api/workspaces/{workspace_id}/loop-runs/{run_id}`); detailed CLI inspection is `loop status --workspace ... --run-id ...`, and the native tools are `compozy__loop_status`/`compozy__loop_validate` after the hard cut.
15. `resolved_runtime` is not present in `loop_generation_outputs`; durable post-reopen status therefore requires an append-only global-stream schema migration and store/DTO/SSE/native/web co-ship in Task 06.
16. The nine upstream skills live under `../compozy/.agents/skills/{cy-*,git-rebase}`. The later optional `cy-capture-decisions` extension is explicitly not added to the v0.3.0 bundle.

## AGH Impact Audit

- **Native tools:** Tasks 02, 03, 06, 08, and 09 own the affected IDs, descriptors, schemas, digests, availability gates, and CLI/API fallbacks; deferred task 14 may extend status/doctor CLI surfaces only. Every other task names the exact checked native-tool surfaces and why they remain unchanged.
- **Extensibility and hooks:** The hard cut covers extension manifests, Host API/MCP namespaces, bundled skills, resources, registries, bridge SDK package identity, and config lifecycle. No new hook taxonomy is introduced; affected descriptors and official references co-ship with their owning task.
- **Workspace data isolation:** Runtime, review, config, skill, and run-status data are classified by scope. Tasks 06-09 (and deferred 14 when unblocked) must propagate `workspace_id` through CLI, HTTP, UDS, core, store, SSE/cache/events, web, and extension calls and prove two-workspace isolation.
- **Official AGH skill:** Public identifiers, tool IDs, loop/config behavior, and migration guidance update the canonical `skills/agh/` content in the same owning task before Task 04 performs the directory/name hard cut to `skills/compozy/`; no task defers a stale public contract to a later sweep.
