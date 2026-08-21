---
status: pending
title: Phase 0 — Foundation Fixes (Global as Concept + Server-Side Catalog Filter)
type: backend
complexity: critical
---

# Task 1: Phase 0 — Foundation Fixes (Global as Concept + Server-Side Catalog Filter)

## Overview

Lands the two prerequisite fixes (D3 + D4) every later task builds on: the home directory stops being a pseudo-workspace (migration `00078` re-homes its work through the four-shape no-workspace disposition, then deletes the row and its boot auto-registration), and session-catalog listing/live updates become server-filtered, fail-closed — the enforcement point profile scoping later extends with a second axis. This task is profile-free by design: no profile concept ships here.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

**Skills**: `eng-code-guidelines`, `golang-master`, `eng-schema-migration`, `eng-test-conventions`, `testing-boss`, `eng-contract-codegen-coship`, `eng-cleanup-failure-paths`.

<requirements>
- MUST implement migration `internal/store/globaldb/schema/migrations/00078_schema.sql` exactly per the `_spec.md` "Phase-0 home-workspace disposition" table: every `REFERENCES workspaces` family enumerated with its fate (rewrite shape 1/2/3, drop-row, assert-empty); a physical `workspace_id` occurrence with no row in that table fails the task, not the operator.
- MUST delete the legacy `~/` workspace row only after the disposition has rewritten everything it owns (its `ON DELETE CASCADE` edges must fire over nothing).
- MUST make assert-empty families abort loudly, naming the family, with nothing partially applied.
- MUST update the declarative schema definitions to the end state in the same change and pass `make codegen-check` (append-only identity: never touch existing migration bytes or `atlas.sum` history).
- MUST delete the boot auto-registration and its cascades (delete targets 1, 2, 7 in `_spec.md` Impact Analysis) — no fallback, no compat shim.
- MUST reject registering the home directory as a workspace with a typed, plain-language error (US-030.AC-3).
- MUST make user-layer resources visible in every workspace lens (US-030.AC-1) — no longer only under the Global view.
- MUST move session-catalog list and stream filtering server-side, fail-closed at subscribe time, replay included (US-031) — as a complete contract slice: spec file → shared `api/core` handler → HTTP and UDS registration → generated types (`make codegen`) → web client switched off client-side filtering (delete target 3).
- MUST NOT introduce any profile concept, column, or vocabulary in this task.
- SHOULD purge the home-workspace clientstate partition via the existing `PurgeWorkspace` path (disposition table, last row).
</requirements>

## Subtasks

- [ ] 1.1 Inventory every physical `workspace_id` occurrence (FK or denormalized) across `internal/store/globaldb/schema/definitions/` and reconcile it against the disposition table before writing SQL.
- [ ] 1.2 Author `00078_schema.sql`: shape-1 NULL rewrites, shape-2 `sessions` (`''` + `scope` column + trigger pair), shape-3 `''`-alone rebuilds (every `length(trim())` CHECK enumerated), drop-row deletions, assert-empty guards, then the `~/` row delete.
- [ ] 1.3 Update declarative definitions to the end state; refresh `atlas.sum` + sqlc output via `make codegen`; pass `make codegen-check`.
- [ ] 1.4 Delete `ensureDefaultWorkspace` + its cascades and `operatorHomeDir`; re-audit `ResolveOperatorHomeDir*` consumers; add the non-registrable guard with its typed error.
- [ ] 1.5 Fix user-layer resource visibility so the user tier composes under every workspace lens.
- [ ] 1.6 Implement the server-side catalog filter: mandatory scope on list queries, subscriber-level predicate on the stream (fail-closed subscribe, filtered replay), params validated at the shared handler, both listeners registered.
- [ ] 1.7 Ship the contract slice: `internal/api/spec/` entries, regenerated TS types, web client consuming server params with client-side filtering deleted.
- [ ] 1.8 Extend the canonical migration suites (fresh/reopen/ahead/integrity/equivalence) with the seeded disposition fixture (IT-003) and per-family assert-empty negatives (IT-075); extend the canonical stream suite with workspace-axis leak coverage (the profile-axis IDs IT-023..025/E2E-011 complete in task_06).
- [ ] 1.9 Update affected docs pages (workspaces registration, Global view semantics) and flag QA scenarios.

## Implementation Details

Migration head is `00077_schema.sql`; this task appends `00078`. The four-shape rule, the per-family fate table, and the fixture demands are specified in `_spec.md` Data Models ("No-workspace representation" + "Phase-0 home-workspace disposition"). The stream fix extends the subscribe/publish seam — no middleware, no context-injected defaults.

### Relevant Files

- `internal/store/globaldb/schema/migrations/` — head `00077_schema.sql`; append `00078_schema.sql` (+ `atlas.sum` refresh via codegen).
- `internal/store/globaldb/schema/definitions/` — 27 files; end-state authority (`20_sessions.sql`, `70_tasks.sql:92-93` shape-1 precedent, `33_extensions.sql:22-43` trigger-pair template, `50_loops.sql`, `60_network.sql`, `37_notifications.sql`).
- `internal/daemon/boot_runtime_foundation.go:133-208` — `ensureDefaultWorkspace` (176-200), overlay special case (133-139), call site (165), `operatorHomeDir` (202-208): the delete site.
- `internal/config/home.go:106-167` — `ResolveOperatorHomeDir*` re-audit; the `.compozy`-suffix fallback dies with its consumer.
- `internal/workspace/resolver_crud.go:13-122` — `Register` gains the home-directory refusal (438 lines — extract, do not append past the cap).
- `internal/session/session_catalog_stream.go:33-96` — `CatalogEvent`, `subscribe` (no filter today), `publish` (fans out to all): the D4 fix site.
- `internal/api/core/session_catalog_stream.go:15-61` + `internal/session/catalog_page.go:41-58` — handler forward + optional-filter leak.
- `internal/api/httpapi/routes.go` + `internal/api/udsapi/routes.go` — hand-mirrored session route registration.
- `internal/api/spec/` — per-domain registry pattern (`registry_sessions.go`) for the new params.
- `internal/config/agent.go:154-211` + `internal/workspace/scanner.go:281-315` — user-layer visibility fix site (discovery roots).
- `web/src/systems/session/hooks/use-session-catalog-streams.ts:40-65` — client-side filtering to delete.

### Dependent Files

- `internal/store/globaldb/queries/*.sql` — queries touching rewritten columns (sessions/tasks/loops/network/notifications) recompile against the new shapes.
- `internal/clientstate/contract.go` — `PurgeWorkspace` for the home partition.
- `web/src/generated/compozy-openapi.d.ts` — regenerated by `make codegen`.
- `magefiles/boundaries.go` — unchanged here, but verify no rule references the deleted boot path.

### Related ADRs

- [ADR-007](adrs/adr-007.md) — the two fixes are phase 0 of this spec, with their own gates.
- [ADR-015](adrs/adr-015.md) — the subscriber-level filter is the enforcement point profiles extend.
- [ADR-013](adrs/adr-013.md) — context: "Global" names only the cross-workspace view after this task.

## Deliverables

- Migration `00078` + updated declarative definitions passing the full fresh/reopen/ahead/integrity/equivalence suites.
- Boot auto-registration deleted; home directory non-registrable with typed refusal.
- User-layer resources visible under every workspace lens.
- Server-filtered session catalog (list + stream + replay) on both transports with regenerated types and the web client off client-side filtering.
- Updated docs pages for workspace registration and the Global view.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] UT-048 — user-layer resources visible under any workspace lens (post-D3).
- [ ] IT-003 — phase-0 disposition equivalence: one seeded fixture with rows in every rewrite/drop family survives `00078` in its labeled shape; `~/` row gone, nothing cascaded.
- [ ] IT-075 — assert-empty negatives: one fixture per assert-empty family; `00078` aborts loudly naming the family.

Note: the stream-filter contract IDs (IT-023..025, E2E-011) and the phase-0 CLI journey (E2E-010) complete in tasks 06 and 04 — this task builds the seam they exercise and extends the canonical stream suite with workspace-axis coverage as part of its gate.

### Web/Docs Impact

- `web/`: `web/src/systems/session/hooks/use-session-catalog-streams.ts` (server params, delete client filter), `web/src/systems/session/hooks/session-catalog-streams-store.ts` (no behavior change expected — verify), `web/src/generated/compozy-openapi.d.ts` (regen).
- `packages/site`: `content/docs/workspaces/` (home no longer registrable; Global = across-workspaces view), `content/docs/configuration/file-locations.mdx` if it references `~/` workspace behavior.
- QA impact: new scenarios — add content-addressed untested files for "home directory registration refused" and "Global view = aggregate across workspaces"; reset any existing workspace-registration scenario touching `~/`. Walk owned by task_13.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked surfaces: extension manifests, hooks, tools/resources registries, bridge SDKs, MCP sidecars; this task changes workspace/topology and stream filtering only.
- Agent manageability: catalog list/stream params are agent-visible contract (structured output; fail-closed errors); `compozy workspace add ~/` refusal is deterministic and typed.
- Config lifecycle: none — no `config.toml` keys added/changed; `ResolveOperatorHomeDir` deletion is code-only (verify no config key references it).
