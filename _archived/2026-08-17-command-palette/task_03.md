---
status: completed
title: "P2 — Personalization, Single TS Scorer, Entity + Settings Search"
type: backend
complexity: high
---

# Task 3: P2 — Personalization, Single TS Scorer, Entity + Settings Search

## Overview

Makes the palette learn and find everything: the personalization store (usage/query-hits/pins tables, fragment `44_cmd_palette.sql` → migration `00070`), the frozen signal routes (`rank-signals`, `usage`, `pins`, `personalization`), and the **single TypeScript scorer** (frecency + adaptive query learning + section assembly) golden-fixture-pinned against the daemon's `WeightsV1`. Completes the root search surface: empty-query pins/recents/curated, ghost autocomplete, and typed entity + settings sections across every list-bearing domain under the one scope-globe model.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting; read every ADR in `adrs/` and any `memory/task_NN.md` present
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — fix root causes; frozen `_dx.md`/Part II contracts are binding; surface gaps as clarifications, never invent
</critical>

<requirements>
1. MUST land fragment `internal/store/globaldb/schema/definitions/44_cmd_palette.sql` → Goose migration `00070` (append-only, after task_01's `00069`): `cmd_palette_usage`, `cmd_palette_query_hits`, `cmd_palette_pins` with the exact columns/PKs of Part II Data Models — side tables, no JSON blobs, `workspace_id` on every row with delete-trigger cascade (SI-7).
2. MUST serve the frozen routes byte-exact per `_dx.md`: `GET /rank-signals` (weights + usage + query_hits + pins + revision; authenticated, workspace-scoped), `POST /usage` (204; normalized query only), `PUT|DELETE /pins/{id}` (idempotent, emits `catalog.changed`), `GET|DELETE /personalization` — on both transports with OpenAPI/TS co-ship.
3. MUST implement scoring as ONE TypeScript module (`web/src/systems/os/lib/ranking/`): pure, headless-tested, consuming the rank-signals snapshot held in session memory only (never IndexedDB, never localStorage); deterministic total order with fixed group precedence, score deadband, and a comparator transitivity property test; every constant lives in daemon-served `WeightsV1` including `fallback_weak_match_threshold` — no magic numbers, no Go scorer (ADR-003 as amended).
4. MUST keep personalization signal maintenance daemon-side: decay math with injected clock, tolerant pruning of stale ids (usage/query-hits/pins/aliases) on read, corruption → zero-signal defaults with one log, `ResetPersonalization` scoped to one workspace; recents derive from `cmd_palette_usage.last_used_at` — no fourth table; "open palette" never recorded.
5. MUST record usage safely: only the normalized pre-selection query — never argument values, never password content (SI-6; the storage-level invariant is IT-008's).
6. MUST implement the empty-query surface (Pinned → Recents → context group → curated; first-run curated, never empty) and ghost autocomplete (high-confidence tail, `→` accepts at end-of-input only, casing preserved).
7. MUST implement entity search client-side over palette-open-gated domain queries (Key Decisions — no new daemon search API) for every list-bearing domain (sessions, tabs, worktrees, agents, tasks, loops, jobs, triggers, bridges, knowledge, vault, network channels, marketplace, extensions) plus settings destinations: typed sections arriving async without reorder/selection theft, per-section inline errors, caps with exact overflow notes, vault names-only enforced at the projected row type, workspace labels under global scope, single globe preference through the existing shared daemon preference.
8. MUST ship the agent surface of this slice: CLI `compozy cmd-palette personalization show|reset` per transcripts + docs; `compozy config get cmd_palette.personalization` reads land with task_05's config (checked — no config key ships here).
9. MUST extend the canonical migrate suites for BOTH migrations (IT-020 completes here: `00069` + `00070` fresh/reopen/ahead/integrity/equivalence + cascade + approval-race rows).
10. MUST block and surface if a cited artboard is missing at execution time.
</requirements>

## Visual Contract

| ID    | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| ----- | -------------------------- | ----------------------------- | -------- | -------- | ---------------------------------- |
| VC-01 | `docs/design/opendesign/command-palette/command-palette-root.html` — rest with Pinned + Recents + context group | ⌘K overlay — workspace with seeded usage/pins | 1440×900 | normative | None |
| VC-02 | `command-palette-root.html` — query with ghost completion tail | ⌘K overlay — 2-3 chars typed, high-confidence top result | 1440×900 | normative | None |
| VC-03 | `command-palette-root.html` — async entity sections resolving | ⌘K overlay — entity query with one domain loading, others resolved | 1440×900 | normative | None |
| VC-04 | `command-palette-root.html` — global-scope rows with workspace labels | ⌘K overlay — globe widened, same-name entities across workspaces | 1440×900 | normative | None |
| VC-05 | `command-palette-root-states.html` — section error state | ⌘K overlay — one domain endpoint failing, inline section error naming the domain | 1440×900 | normative | None |
| VC-06 | `command-palette-root-states.html` — entity section at cap with exact overflow note | ⌘K overlay — one domain seeded at 100× volume | 1440×900 | normative | None |

Evidence for each row: `.compozy/tasks/command-palette/evidence/visual/task_03/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 3.1 Fragment `44_cmd_palette.sql` + migration `00070` + sqlc queries + store wrappers (`global_db_cmd_palette*.go`) + delete-trigger cascade
- [x] 3.2 Daemon signal maintenance: `RecordUsage`/pins/reset on the registry, `DecayFrecency` with injected clock, tolerant prune, corruption degrade, `WeightsV1` (one versioned file)
- [x] 3.3 Frozen routes on both transports (`rank-signals`, `usage`, `pins`, `personalization`) + OpenAPI/TS + parity rows
- [x] 3.4 TS scorer module `web/src/systems/os/lib/ranking/`: match kinds (word-boundary/subsequence/diacritic folding), frecency + query-learning blend, deadband + transitivity, `assembleSections` with promotion floors + golden cross-fixtures pinned to `WeightsV1`
- [x] 3.5 Rank-signals client (session-memory only) + usage reporting wiring from the dispatch seam
- [x] 3.6 Empty-query surface (pins/recents/curated) + ghost autocomplete
- [x] 3.7 Entity search: per-domain adapters/sections over palette-open-gated queries (all list-bearing domains), settings-destination search, scope globe, caps/labels/errors
- [x] 3.8 CLI `cmd-palette personalization show|reset` + transcripts + generated docs
- [x] 3.9 IT-020: extend `migrate_streams_test.go` canonical suites for `00069` + `00070` incl. approval races + cascades
- [ ] 3.10 Visual Contract evidence bundles VC-01..06 — deferred to task_12 by the accepted tail-only QA policy

## Implementation Details

Reference `_spec.md` Part II: Data Models (three tables + rationale), API Endpoints (rank-signals/pins/usage/personalization), Core Interfaces (`Snapshot`/`Weights`/TS module contract), Key Decisions (entity search client-side, weights, catalog caching exclusion of rank signals), Safety Invariants 6–7.

### Skills

`golang-master` · `eng-code-guidelines` · `eng-schema-migration` · `eng-contract-codegen-coship` · `eng-test-conventions` · `testing-boss` · `vitest` · `eng-design` · `ui-craft` · `impeccable` · `eng-ui-screenshot`

### Relevant Files

- `internal/store/globaldb/schema/definitions/33_extensions.sql` + `42_tool_approval_grants.sql` — trigger/FK cascade patterns; `schema/migrations/` (head `00069` after task_01) + `sqlc.yaml` + `global_db_tool_approval_grants.go` — store-wrapper model
- `internal/store/migrate_streams_test.go` — the canonical suite IT-020 extends
- `internal/memory/recall_source.go` + `internal/config/config_memory*.go` — the "daemon owns versioned weights" precedent (BM25 weights)
- `internal/api/core/cmd_palette*.go` + route/spec files from task_01 — the family these routes join
- `web/src/systems/os/lib/ranking/` (new) + `web/src/systems/os/hooks/__tests__/use-os-command-palette.test.tsx` lineage — headless scorer suite home
- `web/src/systems/session/…` sessions/tabs/worktrees search in the task_02 projection — the shipped pattern each domain adapter generalizes
- `web/src/systems/*/lib/query-options.ts` per domain (sessions, tasks, loops, workspace, bridges, network, extensions, …) — the palette-open-gated queries each section consumes
- `web/src/lib/status-tone.ts` + `compareAttentionFirst` — shared dictionaries for entity row badges/ordering
- `internal/cli/window_manager_common.go` + `format.go` — CLI verb/output patterns for `personalization`

### Dependent Files

- `web/src/systems/os/` projection/hooks from task_02 — gain scorer/blend/sections/ghost/empty-query integration
- `openapi/compozy.json` + `web/src/generated/compozy-openapi.d.ts` — regen
- `internal/store/globaldb/schema/migrations/atlas.sum` — `00070` appended
- `packages/site/content/docs/cli/` — personalization verb docs regenerate
- `skills/compozy/` — personalization surface entry (P2 co-ship per Impact Audit)

### Competitor References

- `.resources/supercmd/src/renderer/src/utils/root-search-ranking.ts` — the scoring model to port (match kinds, weights, transitivity comment)
- `.resources/supercmd/src/shared/root-search-ranking-state.ts` — frecency + input-history decay/prune as pure functions
- `.resources/supercmd/src/renderer/src/utils/root-search-sections.ts` — rank→sections with promotion floors
- `.resources/supercmd/src/renderer/src/hooks/useLauncherCommandModel.ts` — empty-query grouping

### Related ADRs

- [ADR-003](adrs/adr-003.md) (as amended) — frecency + query learning daemon-persisted; ONE TS scorer; golden fixtures; deterministic CLI order
- [ADR-006](adrs/adr-006.md) — signals/weights authority daemon-side; volatile evaluation client-side

### Web/Docs Impact

- `web/`: `web/src/systems/os/lib/ranking/` (new), entity-section components/hooks, ghost/empty-query surfaces, rank-signals client, MSW fixtures for the new routes, stories for rest/ghost/sections states.
- `packages/site`: generated CLI docs (`cmd-palette personalization`); no hand-written page this slice (config page lands task_05).
- QA impact: flag only (walks in task_12): reset the **registry-driven root** scenario minted in task_02 back to `untested` (ranking/personalization/entity behavior added — dedup, no new file).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked surfaces: no manifest/hook/SDK change; extension commands rank through the same scorer without special-casing.
- Agent manageability: `GET|DELETE /personalization`, `GET /rank-signals`, `PUT|DELETE /pins/{id}`, `POST /usage` on both transports; CLI `personalization show|reset`; deterministic errors per `_dx.md`.
- Config lifecycle: none this slice — checked: `cmd_palette.personalization` key + section land in task_05; no key added/changed here (store honors the flag once it exists).

## Deliverables

- Migration `00070` + three side tables + store wrappers + IT-020 canonical-suite extension covering both migrations
- Frozen personalization/rank-signal routes live on both transports with OpenAPI/TS co-ship
- Single TS scorer with golden fixtures + transitivity property; zero ranking constants outside `WeightsV1`
- Empty-query surface, ghost, and all-domain entity + settings search under the one globe model
- CLI personalization verbs + generated docs + skills update
- Visual Contract evidence bundles VC-01..06 **(REQUIRED)**
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] UT-020, UT-021, UT-022, UT-023, UT-024, UT-025, UT-026, UT-027, UT-028, UT-029, UT-030 — scorer: match kinds, folding, term-drop, alias-first, context boost, transitivity property, literal metacharacters, frecency decay, prune threshold, query learning, section assembly
- [x] UT-088, UT-089, UT-090, UT-091, UT-092, UT-093, UT-094 — store: upsert, tolerant prune, corruption degrade, workspace isolation, redaction recorder, idempotent pin, scoped reset
- [x] UT-110, UT-111, UT-112, UT-113, UT-114, UT-115, UT-116, UT-117, UT-118, UT-119 — search blend: gated queries, overlong query, vault type-level, caps + notes, wave merge/selection survival, section error, workspace labels, both-groups collision, ghost render/accept rules
- [x] IT-003 — scope preference round-trip across clients
- [x] IT-006 — usage → snapshot loop with injected clock (weights, recents order, pins)
- [x] IT-008 — password-arg command records query only (SQL-level)
- [x] IT-011 — corruption + workspace isolation
- [x] IT-012 — reset scoped to workspace
- [x] IT-020 — migrations `00069`+`00070` canonical suites + approval races + cascades
- [ ] E2E-001, E2E-002, E2E-003, E2E-004, E2E-005, E2E-006, E2E-007 — deferred to task_12 by the accepted tail-only QA policy

## Success Criteria

- Every assigned test case implemented and passing
- Golden fixtures pin scorer output against `WeightsV1` — a weight change breaks the fixture, not silently reorders
- Rank-signals snapshot verifiably absent from IndexedDB/localStorage (asserted in the cache suite)
- Identical query + personalization state → byte-identical order across re-renders (UT-025 + E2E determinism)
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence
- `make gate` green including migrate suites for both migrations
