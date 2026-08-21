# Test Specification: Loop & Task Legibility

Canonical test contract for the loop/task legibility program. Companion to `_spec.md`. Derived from `_user_stories.md` (behavior), `_spec.md` Part II (components), `_dx.md` (CLI/API journeys), `_uiux.md` (browser journeys), and the landed visual contracts at `docs/design/opendesign/loop-legibility/`.

## Strategy

- **Go**: table-driven, `t.Run("Should …")` subtests, `t.Parallel()` default, `-race`/`CGO_ENABLED=1`; fakes at I/O boundaries only; integration uses real SQLite (fresh + reopen) under the `integration` build tag; per `eng-test-conventions`.
- **Web**: Vitest for lib/view-model units (run from repo root via Turbo); Playwright (`make test-e2e-web`) against the daemon-served SPA for browser journeys.
- **Runtime E2E**: Go harness (`make test-e2e-runtime`) with `acpmock`; contract changes co-ship OpenAPI + generated TS + mock matchers in the same change (L-007).
- **Conventions**: status-code AND body assertions on every API case; deterministic error shapes asserted verbatim from `_dx.md`; no `t.Parallel()` on env-mutating tests (L-002).

## Suite Placement

Placement contract (root `CLAUDE.md` test-placement rule): every group below names its invariant, owning layer, and canonical suite, with the reuse decision. One layer owns each invariant; other layers reference it only inside a journey. E2E cases are public cross-boundary journeys, never re-owners of a unit/integration invariant.

| Group (cases) | Invariant | Owning layer | Canonical suite — reuse decision |
| --- | --- | --- | --- |
| Catalog SQL (UT-021..026, IT-011..014) | calm default is a server-owned predicate; provenance projected, never parsed | store | **extend** the existing task-catalog SQL suite in `internal/store/globaldb` (the suite covering `taskCatalogBaseFilter`) — no parallel file |
| Handler mapping (UT-027..029) | `include_loop` maps to neutral kernel fields; deterministic 400s | api/core | **extend** the existing task-catalog param-parsing suite in `internal/api/core` |
| Briefing verdict (UT-001..010, UT-049, UT-051) | served verdict is the page's truth (cascade, tones, usage, terminal outcome/artifacts) | internal/loop | **new** canonical suite `internal/loop/briefing_test.go` — new behavior, no existing owner |
| Timeline view map (UT-011..013) | classification exhaustive; positions deterministic | internal/loop | **new** `internal/loop/timeline_view_test.go` |
| Roster assembly (UT-014..020, UT-048, UT-050) | total source→projection mapping; `not_taken` only from route evidence; filter allowlist = `_dx.md` | internal/loop | **new** `internal/loop/roster_test.go` |
| Settlement + sweep (UT-030..033, IT-001..010, IT-028..029) | one settlement authority; boot barrier precedes claims; idempotent repair | store+loop (integration) | **extend** the existing settlement/rollup integration suite in `internal/store/globaldb`; the transition-path coverage test lives beside `settleLoopRunTerminal`; sweep/barrier get one co-located integration file |
| Surface parity (IT-015..016, IT-026) | same default + scoping on all four surfaces | api (integration) | **extend** the existing API parity/testutil harness pattern in `internal/api` |
| Read-layer integration (IT-017..023, IT-027) | pagination stability, cursor identity, view union | api+loop (integration) | co-located with the new routes' handler suites — **extend** the loops handler test files |
| Config (IT-024) | key lifecycle validation | internal/config + daemon | **extend** the existing config validation suite |
| Web view-models (UT-035..047) | view-model truth per lib | web (Vitest, co-located) | **extend** the co-located lib suites; suites for deleted code (`task-formatters` regex) are deleted with it — regressions move to the provenance renderer's suite |
| Browser journeys (E2E-010..020) | user-visible flows only | e2e-web (Playwright) | **extend** the canonical tasks spec; the redesigned run page gets one canonical loop-run spec |
| Runtime journeys (E2E-001..006) | public CLI/daemon journeys | e2e-runtime (Go harness) | **extend** the existing loop e2e harness suites |
| Default-read/briefing-test visual behavior | SD-012 register assertions | QA walks (`docs/qa/scenarios/`) + Visual Contract rows in task_04/task_05 against `docs/design/opendesign/loop-legibility/` | owned by scenario walks and `eng-ui-screenshot` bundles — deliberately NOT frozen in unit tests (L-036) |

## Coverage Matrix

| Source | Behavior | Unit | Integration | E2E |
| --- | --- | --- | --- | --- |
| US-001 | listings show work items only | UT-021 | IT-011 | E2E-001, E2E-010 |
| US-001.EC-1 | cells most recent, every page clean | — | IT-011 | — |
| US-001.EC-2 | loop-only workspace → true empty | UT-022 | — | E2E-010 |
| US-001.EC-3 | cross-workspace never visible | — | IT-016 | — |
| US-001.EC-4 | provenance, never name matching | UT-023 | — | — |
| US-002 | opt-in reveal, distinguished + linked | UT-024, UT-040 | IT-012 | E2E-011 |
| US-002.EC-1 | reveal-empty truthful message | UT-041 | — | E2E-011 |
| US-002.EC-2 | run retention-deleted → truthful degrade | UT-042 | IT-033 | E2E-020 |
| US-002.EC-3 | other-workspace records never revealed | — | IT-016 | — |
| US-003 | aggregates count work items only | — | IT-014 | E2E-010 |
| US-003.EC-1 | loop-only escalations → inbox empty, bell lane holds | — | IT-014 | E2E-010 |
| US-003.EC-2 | mixed escalations split correctly | — | IT-014 | — |
| US-004 | structured listing parity + provenance fields | UT-027 | IT-015 | E2E-001 |
| US-004.EC-1 | `--parent` drill-down without global flag | — | IT-026 | — |
| US-004.EC-2 | cursor binds filter context | UT-026 | IT-011 | — |
| US-004.EC-3 | no compat shim; skill documents the contract | — | — | E2E-006 (docs assertion in QA walk) |
| US-005 | 30s briefing default read | UT-001..
UT-006 | IT-020 | E2E-012 |
| US-005.EC-1 | queued/admission-parked reads plainly | UT-007 | — | E2E-012 |
| US-005.EC-2 | watching/dormant calm | UT-008 | — | — |
| US-005.EC-3 | deep link cross-workspace scoped | — | IT-016 | — |
| US-006 | steps + rounds progress | UT-015, UT-035 | IT-017 | E2E-012 |
| US-006.EC-1 | single-pass hides round counter | UT-036 | — | — |
| US-006.EC-2 | not-taken excluded from totals | UT-016, UT-037 | IT-019 | — |
| US-006.EC-3 | all-parked round states reason | UT-038 | — | — |
| US-006.EC-4 | control-only segments contribute no steps | UT-035 | — | — |
| US-007 | needs-you unmissable + actionable | UT-002 | — | E2E-013 |
| US-007.EC-1 | multiple needs-you ordered with count | UT-003 | — | E2E-013 |
| US-007.EC-2 | resolved elsewhere → resolves live | — | IT-027 | E2E-013 |
| US-007.EC-3 | expiry stated, never auto-retry | UT-004 | — | — |
| US-008 | outcome + artifacts lead terminal | UT-005, UT-051 | — | E2E-014 |
| US-008.EC-1 | no-op states plainly | UT-006 | — | — |
| US-008.EC-2 | canceled shows actor + time | UT-005 | IT-002 | E2E-014 |
| US-008.EC-3 | pruned blob truthful note | UT-043 | — | — |
| US-009 | narrated durable story | UT-011, UT-044 | IT-021 | E2E-015 |
| US-009.EC-1 | thousands of beats paged responsive | — | IT-021 | E2E-015 |
| US-009.EC-2 | two windows converge | — | IT-021 | — |
| US-009.EC-3 | fork beat links related run | UT-012 | IT-022 | — |
| US-010 | roster answers "which need me" | UT-045 | IT-032 | E2E-018 |
| US-010.EC-1 | empty roster explains start | UT-045 | — | E2E-018 |
| US-010.EC-2 | needs-you never sorts below terminal (server-ordered) | UT-045 | IT-032 | E2E-018 |
| US-011 | live DAG per-node state | UT-014, UT-046 | IT-017 | E2E-016 |
| US-011.EC-1 | 100-item fan-out stays a rollup | UT-017 | IT-018 | E2E-016 |
| US-011.EC-2 | not-taken neutral ≠ failure | UT-016 | IT-019 | E2E-016 |
| US-011.EC-3 | deep graph locatable | — | — | E2E-016 |
| US-011.EC-4 | terminal graph faithful | — | IT-017 | E2E-016 |
| US-012 | complete roster, healthy included | UT-014 | IT-017 | E2E-017 |
| US-012.EC-1 | zero-action-node run truthful | UT-020 | — | — |
| US-012.EC-2 | strategy-cancel ≠ operator-cancel | UT-018 | — | E2E-017 |
| US-012.EC-3 | fan-out grouped with rollup | UT-017 | IT-018 | — |
| US-013 | generation history + outcomes | UT-019 | IT-017 | E2E-017 |
| US-013.EC-1 | crash-interrupted round true partial | — | IT-005 | — |
| US-013.EC-2 | no scoring → no invented columns | UT-019 | — | — |
| US-014 | node verbs from new views | — | IT-027 | E2E-017 |
| US-014.EC-1 | concurrent interventions single winner | — | IT-008 | — |
| US-014.EC-2 | stale-view intervention re-validated | — | IT-027 | — |
| US-015 | bidirectional node↔session↔record links | UT-047 | — | E2E-016, E2E-020 |
| US-015.EC-1 | pruned session truthful degrade | UT-047 | — | — |
| US-015.EC-2 | never-started node: no fabricated links | UT-016 | — | — |
| US-016 | terminal ⇒ records settled, all paths | — | IT-001..IT-004 | E2E-001 |
| US-016.EC-1 | crash window converges ≤1 cycle | — | IT-005 | E2E-002 |
| US-016.EC-2 | sweep vs live settle single winner | — | IT-008 | — |
| US-016.EC-3 | already-settled → sweep no-op | UT-030 | IT-006 | — |
| US-017 | retro repair on first boot | — | IT-005, IT-010 | E2E-002 |
| US-017.EC-1 | run row missing → run_missing reason | UT-032 | IT-009 | — |
| US-017.EC-2 | eligibility removed before claims; backlog non-blocking | — | IT-028, IT-005 | — |
| US-018 | reconciliation auditable + distinct | UT-031 | IT-025 | E2E-002 |
| US-018.EC-1 | reason filterable/structured | — | IT-025 | — |
| US-019 | structured roster/timeline reads | UT-014 | IT-017, IT-021 | E2E-005 |
| US-019.AC-4 | briefing parity for agents | — | IT-020 | E2E-004 |
| US-019.EC-1 | pagination stable under appends | — | IT-018, IT-021 | — |
| US-019.EC-2 | cross-workspace denied not empty | — | IT-016 | — |
| US-019.EC-3 | huge roster paged + rollups | — | IT-018 | — |
| US-020 | events follow with resume | — | IT-021, IT-023 | E2E-003 |
| US-020.EC-1 | no events yet → waits quietly | — | — | E2E-003 |
| US-020.EC-2 | beyond-head + foreign-cursor deterministic errors | UT-013 | IT-022 | E2E-006 |
| Catalog classification (Part II) | predicate/join/projection | UT-021..UT-026 | IT-011..IT-015 | — |
| Handler mapping | include_loop → neutral fields | UT-027..UT-029 | IT-015 | — |
| Briefing fn | verdict cascade + current-state truth | UT-001..UT-010, UT-049 | IT-020 | — |
| View map | notable/activity/chatter exhaustive; positions | UT-011..UT-013 | IT-023 | — |
| Roster assembly | total mapping; pending ≠ not_taken | UT-014..UT-020, UT-048 | IT-017..IT-019 | — |
| Sweep + barrier + backfill | guards/reasons/barrier fail-closed/provenance | UT-030..UT-033 | IT-005..IT-010, IT-028, IT-029, IT-031 | E2E-002 |
| Runs-list summary/ordering | server-owned S6 read | — | IT-032 | E2E-018 |
| Single-task provenance | shared LoopProvenance on detail | — | IT-033 | E2E-020 |
| Settlement primitive | cause matrix; children-first; no-bypass coverage | — | IT-001..IT-004, IT-030 | — |
| Config | reconcile_interval lifecycle | — | IT-024 | — |
| Web tasks libs | reveal/provenance/empty states | UT-040..UT-042 | — | E2E-010, E2E-011 |
| Web run-page libs | story/usage/links view-models | UT-043..UT-047 | — | E2E-012..E2E-019 |

## Unit Tests

### Briefing verdict cascade (Part II: Run read layer) — `internal/loop`

- **UT-049** (state, N-004): node failed on attempt 1 and recovered on attempt 2 while the run continues — the briefing reads `tone=ok` with the current activity as headline; the absorbed failure never wins the headline (current-state truth); attempt history remains visible via the roster (cross-checked with UT-014).
- **UT-051** (state, B-002): terminal briefings carry typed results — `done` populates `outcome{status,cause,at}` + `artifacts[]` with `availability=available`; a partial failure yields `availability=partial` labeling; a pruned blob yields `availability=pruned` with the name retained; canceled/killed populate `outcome.actor_kind/actor_ref`; the human "Produced: …" line is derived from these fields, never the reverse.

- **UT-001** (happy): running run, no blockers — returns `tone=ok`, headline names the running step and round, `progress={round,steps_done,steps_total}` matches roster, `usage` populated.
- **UT-002** (state): open approval gate — `tone=needs_you`, blocker `kind=approval` with `gate_id`, `waiting_since`, `unblocker` string exactly `compozy loop approve <run> --gate <id>`.
- **UT-003** (ordering): approval + quarantined node + open request simultaneously — blockers ordered approval, quarantine, request; headline reflects the first; count = 3.
- **UT-004** (state): expired request — blocker carries expiry fact; no retry/auto-retry field emitted.
- **UT-005** (state): terminal `canceled` — headline includes actor kind and timestamp; `tone=failed` only for `failed/exhausted/stalled`; canceled reads neutral.
- **UT-006** (state): terminal `no-op` — headline states no outputs produced; artifacts list empty, not fabricated.
- **UT-007** (state): queued on admission (concurrency cap) — `tone=ok`, headline "waiting to start" with the cap reason.
- **UT-008** (state): `watching` dormant — `tone=ok`, headline names what it waits for; no activity implied.
- **UT-009** (error): unknown run id — service returns the typed not-found error (handler maps to `404 loop_run_not_found`).
- **UT-010** (boundary): run with zero generations and zero requests — verdict still non-empty (`ok`/queued), never a nil briefing.

### Timeline view map (Part II: Run read layer) — `internal/loop`

- **UT-011** (happy): every kind in the `loop_run_events` CHECK list maps to exactly one of `notable|activity|chatter`; the mapping is exhaustive — the test enumerates the CHECK list and fails on any unclassified kind (classification is a same-change requirement for new kinds; no runtime default exists). `all` = ordered union of the three tiers.
- **UT-012** (state): `run_forked` entry renders a `notable` title carrying the related run id; a coalesced heartbeat-class entry spans `first..last` and carries `seq = last` (resume after it replays nothing).
- **UT-013** (error): HTTP page-cursor decoding — opaque cursor from run B applied to run A returns the typed branch-changed error; malformed cursor returns the typed invalid-cursor error; a CLI-style numeric position beyond the run's head returns the typed beyond-head error naming the head.

### Roster assembly (Part II: Run read layer) — `internal/loop`

- **UT-014** (happy): run with succeeded/running/queued nodes — every authored node×generation present with state, attempt, timing, `session_id`, `cell_task_id`.
- **UT-015** (happy): steps progress derives from roster — settled action nodes / total action nodes for the current generation; control nodes contribute zero.
- **UT-016** (state): route-not-taken branch — authored node with **durable route evidence against it** (`route_taken` electing another arm, or `branch_pruned`) renders `state=not_taken`, no timestamps, no links.
- **UT-048** (state): downstream authored node with **no output row and no route evidence** renders `state=pending` (reachable, upstream unsettled) — never `not_taken`; it appears under `state=all` and is excluded by `state=running`. (`pending` is an output state, not a public filter value — B-007.)
- **UT-050** (boundary): the roster `state` filter allowlist equals the `_dx.md` vocabulary exactly (`all|running|queued|waiting|retrying|paused|quarantined|succeeded|failed|canceled|not_taken`); any other value (incl. `pending`) returns the typed invalid-node-state error listing the allowed set.
- **UT-017** (boundary): fan-out 100 items — `fanout_rollups` carries `{done,total,failed}`; default page returns rollup + first page of items under `next_cursor`.
- **UT-018** (state): strategy-canceled node carries the strategy cause; operator-canceled carries actor — the two render distinct dispositions.
- **UT-019** (happy): generation history — per-round outcomes and verdicts when the loop defines scoring; absent scoring yields no score fields.
- **UT-020** (boundary): run terminal before generation 1 — roster returns empty node list with the run's terminal state, not an error.

### Catalog classification (Part II: Catalog) — `internal/store/globaldb` + `internal/task`

- **UT-021** (happy): `ExcludeCreatedBy=[{daemon,loop-coordinator}]` — coordinator and cell rows excluded; operator rows (incl. daemon-created non-loop) remain.
- **UT-022** (boundary): workspace containing only loop records — filtered result is empty with zero-count facets (true empty).
- **UT-023** (state): user task titled "loop.deploy coordinator" with `created_by={operator,…}` — always included (provenance, never name).
- **UT-024** (happy): include path — cells project `loop{run_id,loop_name,role=cell,generation,node_id,item_index}` from metadata; coordinator projects `role=coordinator` (post-backfill metadata).
- **UT-025** (happy): `LoopRunID` scoping joins `task_runs.loop_run_id` — returns that run's coordinator + cells only.
- **UT-026** (state): cursor issued with the exclusion filter, replayed with include — deterministic documented behavior (cursor context honored; no row duplication/skip).

### Handler mapping (Part II: API) — `internal/api/core`

- **UT-027** (happy): `include_loop=true` clears `ExcludeCreatedBy`; absent/false sets the daemon/loop-coordinator exclusion; `internal/task` never receives the literal "loop-coordinator" from anywhere else (constant owned by handler layer wiring).
- **UT-028** (happy): `loop_run_id=X` implies include and sets `CatalogQuery.LoopRunID=X`.
- **UT-029** (error): `include_loop=banana` → `400 {"error":"invalid_query_field","field":"include_loop"}` (status AND body).

### Reconciliation sweep guards (Part II: Settlement) — `internal/loop`

- **UT-030** (idempotency): record already settled — sweep writes nothing, emits no event, `SweepReport.RecordsSettled=0`.
- **UT-031** (happy): orphan of a terminal run — settled with reason `reconciled_run_terminal`, distinct from inline `loop_run_terminal`.
- **UT-032** (state): orphan whose `loop_runs` row is gone — settled with reason `run_missing`.
- **UT-033** (state): coordinator task lacking metadata — `BackfillProvenance` backfills `loop_run_id/loop_name/workspace_id` and returns the repaired count; a second pass returns `0` (idempotent). The monitoring key `provenance_backfilled` logs this same return value (N-002: one owner).

### Web view-model units (Vitest) — `web/src/systems/{tasks,loops}`

- **UT-035** (happy): steps-progress view-model renders "step 3 of 6" from briefing progress; control-only rounds render no step gap.
- **UT-036** (boundary): round 1 single-pass — round counter absent from the rendered label.
- **UT-037** (state): not-taken branches excluded from step totals and rendered with the neutral glyph, never danger.
- **UT-038** (state): all action steps parked — progress label states the dominant park reason, no frozen percentage.
- **UT-040** (happy): revealed loop row renders plain identity ("revisao-paralela · round 1 · step revisor-perf"), loop glyph, and href to `/loop-runs/<id>` — no raw task id as primary text.
- **UT-041** (state): reveal-on with zero loop records — renders the filter-scoped empty message, not the generic empty state.
- **UT-042** (state): revealed record whose run link resolves 404 — renders "run no longer available" degrade.
- **UT-043** (state): artifact entry with pruned blob — renders "content no longer stored" note, entry retained.
- **UT-044** (happy): story beat mapper — timeline entry `{kind:node_failed,…}` renders a meaning title from the label map; no enum text leaks; heartbeat-class entries coalesce to one `×N` row.
- **UT-045** (happy/ordering): runs-roster view-model groups needs-you → active → terminal; empty roster yields the start-a-run empty state.
- **UT-046** (state): DAG node chip pairs icon+text for every state (color never sole carrier — asserts label text present per state).
- **UT-047** (state): node panel links — session link rendered from roster `session_id` post-terminal; pruned session renders truthful degrade; not-taken node renders no links.

## Integration Tests

### Terminal settlement on every path (US-016) — `integration` tag, real SQLite

- **IT-001**: natural completion — run reaches `done` via `settleLoopRunTerminal`; same-transaction result per the cause matrix: coordinator `completed`, any live leftover cell `canceled` (reason detail "run done"), already-terminal cells untouched, children before coordinator, zero live records; task events carry `reason=loop_run_terminal`.
- **IT-002**: `POST …/cancel` — run `canceled`; coordinator and live cells become `canceled` (never `completed`) in the same transition; audit actor recorded.
- **IT-003**: `POST …/kill` — coordinator + live cells `canceled` per the matrix; `compozy task list --include-loop --loop-run <id>` shows all settled with truthful statuses (matches the `_dx.md` kill transcript).
- **IT-004**: child-loop stop path (`applyCoordinatorRunStopsWithExecutor`) — child run terminal (`failed`) ⇒ child's coordinator `failed`, live cells `canceled` with the run outcome in the reason, terminal cells untouched — all via the same primitive.
- **IT-030** (coverage): transition-path coverage — enumerates every code path that writes a terminal loop-run status and fails if any of them does not call `settleLoopRunTerminal` in-transaction (the matrix analogue of the observability coverage test).
- **IT-005**: crash simulation — write run terminal, skip settlement, restart harness daemon: boot sweep settles within one cycle; readiness endpoint healthy before backlog completes.
- **IT-006** (idempotency): run sweep twice over the same repaired state — second `SweepReport` all zeros; `task_events` count unchanged.
- **IT-007**: active (running) run with live records — sweep leaves every record untouched.
- **IT-008** (concurrency): sweep and inline settlement race the same run under `-race` — exactly one settlement event per record; final state identical regardless of winner.
- **IT-009**: orphaned records with deleted `loop_runs` row — settled with `reason=run_missing`.
- **IT-010**: pre-existing coordinator without metadata (terminal run) — first boot's provenance pass backfills; catalog include-path then projects `role=coordinator`.
- **IT-028** (concurrency, B-004): boot barrier vs claim — seed terminal-run orphans with queued/claimable runs; start the daemon under `-race`; assert `NeutralizeOrphans` completes before any claimer starts and `ClaimNextRun` never returns a run owned by a terminal or missing loop run (zero stale claims across N racing claim attempts).
- **IT-031** (error, B-005): boot barrier fails closed — inject a store error into `NeutralizeOrphans`; assert readiness is never reported, zero claim attempts occur, no recovery/backstop/ticker starts, and a structured startup error is surfaced; a subsequent successful boot completes neutralization and proceeds normally.
- **IT-032** (ordering, B-001): extended runs list — seed a needs-you run, several active runs, and terminal runs spanning multiple pages; assert server ordering needs_you → active → terminal holds across page boundaries, `attention`/`progress` fields are correct on every surface (HTTP/UDS/CLI), and cursors stay stable.
- **IT-033** (state, B-003): single-task read — `GET /api/tasks/:id` for a cell and a coordinator returns the same `loop` provenance object as the catalog item (deep link independent of list navigation); a retention-deleted run's record omits `loop_name` and keeps `run_id`/`role` (relational facts only).
- **IT-029** (state, B-005): provenance backfill on an ACTIVE run's pre-change coordinator — metadata backfilled (metadata-only write; status/runs untouched — lifecycle invariant intact); and a record whose run row is gone renders the truthful degraded `loop` shape (`run_id` + `role` from relational facts, no `loop_name`, no string parsing).

### Catalog surfaces (US-001..004) — `integration` tag

- **IT-011**: active run minting cells as most-recent rows — default `GET /api/tasks` page 1..N contain zero loop records at limit 50, sort recent; counts/facets match visible rows.
- **IT-012**: `include_loop=true` — coordinator + cells returned with correct `loop` objects (role, generation, node, item).
- **IT-013**: `loop_run_id=<run>` — exactly that run's records (coordinator included) across HTTP and UDS; seeds one cell with **three attempt runs** and asserts exactly one item for it, coherent facets/counts, and gap-free cursors (correlated `EXISTS` semi-join, applied before ordering/pagination — B-006).
- **IT-014**: dashboard aggregates + inbox lanes — status breakdowns exclude loop records; quarantined cell absent from inbox while the loop attention probe still reports it.
- **IT-015** (parity): same query via HTTP, UDS, CLI `-o json`, and `compozy__task_list` — identical item sets and provenance fields; `include_loop` default false on all four.
- **IT-016** (permissions): run/roster/briefing/timeline/catalog queries against another workspace's ids — `404 loop_run_not_found` / scoped-empty catalog; body asserted; never data.
- **IT-026**: `parent_task_id=<coordinator>` without `include_loop` — returns the cells (explicit parentage wins).

### Run read layer (US-011..013, US-019) — `integration` tag

- **IT-017**: live run mid-generation — `GET …/nodes?state=all` returns every authored node×generation; states agree with run detail and with post-terminal re-read (terminal graph faithful).
- **IT-018**: fan-out 100 — pagination stable while items settle concurrently (no skip/dup across pages); rollups correct on every page.
- **IT-019**: route with untaken branch — `state=not_taken` rows present; `state=running` filter excludes them.
- **IT-020**: `GET …/briefing` equals the internal verdict for identical state (served-verdict parity); usage fields match run accounting.
- **IT-021**: timeline — append events while paging; pages gap-free/duplicate-free; `--after <seq>` resume returns exactly the missed entries; two concurrent readers converge on identical order.
- **IT-022**: opaque HTTP page cursor minted for run A replayed on run B (and on A's fork, which is a new run id) — `409 {"error":"timeline_branch_changed"}`; body asserted. (CLI `--after` has no cross-run form — its beyond-head error is E2E-006.)
- **IT-023**: `view=notable` excludes chatter kinds; `view=all` returns them; counts match the classification map.
- **IT-027**: node verb via API while roster/SSE observed — requeue reflects in roster read ≤ one poll cycle; stale-state verb returns the existing deterministic conflict.

### Config lifecycle (Part II: Config)

- **IT-024**: `[loops] reconcile_interval` — default `1m` effective when absent; `"30s"` honored; `"0s"` fails validation with `reconcile_interval must be positive`; boot sweep runs even with large interval.

### Audit read (US-018)

- **IT-025**: `GET /api/tasks/:id/timeline` on a swept coordinator — the settlement event carries structured `reason=reconciled_run_terminal` as a machine-readable field; a naturally settled task shows `loop_run_terminal`; a retention-orphan shows `run_missing`. (No cross-task reason-filtered audit query exists — B-007.)

## End-to-End Tests

### Runtime (Go harness, `make test-e2e-runtime`)

- **E2E-001** (US-001, US-004, US-016): `_dx.md` golden path verbatim — start run, `task list` clean during run, `loop why` healthy → approval → approve → run completes → `task list --include-loop --loop-run` all settled.
- **E2E-002** (US-016, US-017, US-018): orphan repair — seed two terminal runs with live coordinators (the 2026-08-19 incident shape), boot daemon: both settle with `reconciled_run_terminal`; second boot emits nothing.
- **E2E-003** (US-020): `loop events --follow -o jsonl` — kill connection mid-run, resume `--after <last seq>`: zero gaps/dups; stream ends exit 0 at terminal; follow on eventless run waits silently until first event.
- **E2E-004** (US-019.AC-4, US-007): `loop why -o json` during open gate — execute the returned `unblocker` string verbatim; run proceeds; subsequent `why` drops the blocker.
- **E2E-005** (US-012, US-019): force one node to fail once then recover — `loop nodes --run --all -o json` shows `attempt=2`, `attempts[]` with both entries, state `succeeded` (current-state truth).
- **E2E-006** (errors): unknown run → exit 1 + message from `_dx.md`; `--state running` without `--run` → exit 2 + guard message; `--after` beyond the run's head → exit 1 with the beyond-head message naming the current head.

### Web (Playwright, `make test-e2e-web`)

- **E2E-010** (US-001, US-003): with an active loop — Tasks list/kanban show zero loop rows, coherent group counts; dashboard numbers exclude cells; loop-only workspace shows true empty state.
- **E2E-011** (US-002): enable reveal filter — distinguished rows appear; navigating away and back resets the filter; clicking a cell lands on `/loop-runs/<id>`; reveal-empty message when no records.
- **E2E-012** (US-005, US-006): run page during execution — briefing strip, needs-you area, "step N of M · round R", usage (tokens · cost · budget) all visible without disclosure; no `loop.`/`looprun-` ids in the default read (DOM assertion).
- **E2E-013** (US-007): approval flow — card readable (action, asker, choices), approve in place resolves it; two simultaneous needs-you render ordered with count; resolving one keeps the other; CLI-resolved request clears live.
- **E2E-014** (US-008): failed run — failure signal + plain cause visible with all disclosures collapsed; done run leads with outcome + artifact links; canceled shows actor.
- **E2E-015** (US-009): long run (>500 events) — reload, scroll story to run start: full history pages in; no missing tail.
- **E2E-016** (US-011, US-015): operator register — DAG shows running (edge pulse), succeeded, failed, not-taken (neutral), fan-out rollup chip; node click opens panel; session + record links work on a terminal run.
- **E2E-017** (US-012, US-013, US-014): roster lists healthy + parked nodes with attempts and tokens/cost; generation history shows per-round outcome + usage; requeue from the node panel updates state live.
- **E2E-018** (US-010): runs roster — needs-you run renders first and distinct; plain outcome labels; empty state present on fresh workspace.
- **E2E-019** (accessibility): `prefers-reduced-motion` — edge pulse absent (unmounted, not paused); every state chip carries text+icon (no color-only assertion via accessible names).
- **E2E-020** (US-002.EC-2, US-015): task detail of a revealed cell — provenance block + "Open run" works; with run deleted, truthful "run no longer available".
