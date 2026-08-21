---
status: in_progress
title: "Loop run read layer: roster, briefing, timeline & CLI verbs"
type: backend
complexity: high
---

# Task 3: Loop run read layer: roster, briefing, timeline & CLI verbs

## Overview

Delivers front 2: the three computed projections — node×generation roster, pure-function briefing, durable paged timeline — over the existing loop tables, exposed as HTTP/UDS routes and the CLI verbs `loop why`, `loop nodes --run --all`, `loop events --after/--follow/--view`, plus the server-owned runs-list extension (`attention` + `progress` + pre-pagination ordering). This is the single source of truth the web two-register redesign (task_05) and agents both read; it also carries the `_dx.md` golden-path E2E verbatim.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST implement `RunReadService` (`NodeRoster`, `Briefing`, `Timeline`) in `internal/loop` as computed projections — no writes, no materialized tables (ADR-005); new single-responsibility files under the 500-line cap; `internal/loop` never imports `internal/api/*`/`internal/daemon`.
2. MUST implement the roster projection contract exactly: closed enum = the 14 persisted output states plus derived `not_taken`; `not_taken` derives ONLY from durable route evidence (`route_taken` electing another arm, `branch_pruned`, recorded route causes); a reachable node with no output row projects `pending`; controls/waits/retry-schedule overlay precedence; the source→projection mapping is total and exhaustively unit-tested (Safety Invariant 14). Nodes carry `attempts[]` (first public exposure of `loop_node_attempts`), `next_retry_at`, `child_loop_run_id`, structured cancellation disposition/actor, `session_id`, `cell_task_id`, optional truthful `usage`.
3. MUST implement the briefing as a pure deterministic function over roster + requests/gates + run status + usage: cascade `approval > quarantine > request > failure > backoff/quota > running > terminal`; tones `ok|needs_you|degraded|failed`; blockers carry the exact unblocker command; terminal briefings populate typed `outcome{status,cause,actor_kind?,actor_ref?,at}` + `artifacts[]{name,output,ref,availability: available|partial|pruned}` (B-002) — human text is a projection of the typed fields; current-state truth: an absorbed failure never wins the headline (UT-049).
4. MUST implement the timeline as a durable paged read over `loop_run_events`: first page = newest window carrying `head_seq` (0 for event-less runs); older pages snapshot-fenced backward via the opaque cursor `{run_id, view, fixed_head_seq, before_seq}`; cursor replay against another run/fork → `409 timeline_branch_changed`; the notable/activity/chatter map is authored Go code, exhaustive over the event-kind CHECK list (test fails on unclassified kinds); `view=all` = ordered union; heartbeat-class runs coalesce server-side with `seq = last` (Safety Invariant 7).
5. MUST keep two position types deliberately distinct: the opaque HTTP page cursor vs CLI `--after <seq>` (plain per-run sequence; beyond-head → deterministic error naming the head; no cross-run token at the CLI).
6. MUST extend the existing loop-runs list read: items gain `attention?{kind,count,since}` + `progress{round,steps_done,steps_total}`, with needs_you → active → terminal ordering applied server-side BEFORE pagination — same fields on HTTP, UDS, and `compozy loop runs` (B-001; MVP operational summary, not cross-run analytics).
7. MUST register the three new routes (`/nodes`, `/briefing`, `/timeline`) on HTTP AND UDS in the same change, workspace-scoped at the query layer (cross-workspace → `404 loop_run_not_found`, never data); the existing SSE stream stays unchanged as the push channel; live readers attach with `after_sequence=head_seq` after one first page (durable→live seam, de-duped by monotonic seq).
8. MUST ship the CLI verbs per `_dx.md` verbatim (transcripts, exit codes, and the Errors table are the contract): `loop why` (+ `-o json` with outcome/artifacts), `loop nodes --run --all [--state|--generation]` (state allowlist = the `_dx.md` vocabulary exactly — `pending` is an output state, not a filter value, UT-050), `loop events [--after --follow --view] (-o jsonl)` with clean terminal exit, `loop runs` extended output.
9. MUST serve the briefing verdict the page renders — web never re-derives (Safety Invariant 12); existing redaction rules apply to every new payload (Safety Invariant 13).
10. MUST co-ship the contract: OpenAPI authoring + `make codegen` (generated TS) + E2E fixtures/matchers (L-007); update the official Compozy skill loops reference (`loop why/nodes/events` + settlement audit reasons vocabulary).
11. SHOULD reuse the existing inventory/route-cause queries (`ListLoopRouteCauses`, node inventory projections) before authoring new SQL.
</requirements>

## Subtasks

- [ ] 3.1 Roster assembly in `internal/loop` (state projection + precedence + `not_taken`/`pending` split + attempts + fan-out rollups + pagination)
- [ ] 3.2 Briefing pure function (cascade, tones, blockers+unblockers, typed outcome/artifacts, progress, usage)
- [ ] 3.3 Timeline read (view map exhaustive over the CHECK list, coalescing, `head_seq` + snapshot-fenced backward cursors, 409 branch-changed)
- [ ] 3.4 Runs-list extension (`attention`/`progress`, pre-pagination ordering) across HTTP/UDS/CLI
- [ ] 3.5 Contract payloads + route registration (HTTP + UDS, same change) + workspace scoping
- [ ] 3.6 CLI verbs: `loop why`, `loop nodes --run --all`, `loop events --after --follow --view`, `loop runs` output — table + structured output per `_dx.md`
- [ ] 3.7 Durable→live seam for `--follow` (first page + SSE `after_sequence=head_seq`, seq de-dupe, clean terminal exit)
- [ ] 3.8 Contract co-ship: OpenAPI, `make codegen`, generated TS, E2E fixtures/matchers
- [ ] 3.9 Official skill loops reference update (`skills/compozy/`)
- [ ] 3.10 Implement assigned unit tests (UT-001..020, UT-048..051)
- [ ] 3.11 Implement assigned integration tests (IT-017..023, IT-027, IT-032)
- [ ] 3.12 Implement assigned runtime E2E (E2E-001, E2E-003..006)
- [ ] 3.13 Flag QA scenarios per the QA impact line

## Implementation Details

Follow `_spec.md` Part II: Core Interfaces (`RunReadService`, `Briefing`), Data Models (roster projection contract; `LoopRunNodesResponse`/`LoopBriefingResponse`/`LoopTimelineResponse`/`LoopRunListItem` rows), API Endpoints (timeline pagination + handoff semantics), Safety Invariants 7, 10, 12-14. Skills: `eng-code-guidelines` + `golang-master` + `eng-contract-codegen-coship`; tests per `eng-test-conventions` + `testing-boss` + `eng-consolidate-test-suites`; completion per `deslop` + `cy-final-verify`.

Suite placement (from `_tests.md`): briefing/view-map/roster get NEW canonical suites in `internal/loop` (`briefing_test.go`, `timeline_view_test.go`, `roster_test.go` — new behavior, no existing owner); read-layer integration EXTENDS the loops handler test files; runtime journeys EXTEND the existing loop e2e harness suites.

### Relevant Files

- `internal/store/globaldb/schema/definitions/50_loops.sql` — `loop_runs:539`, `loop_generations:63`, `loop_generation_outputs:76`, `loop_node_controls:101`, `loop_node_attempts:135`, `loop_node_waits:158`, `loop_run_events:517` — the projection sources.
- `internal/store/globaldb/queries/loop_core.sql` — `ListLoopRunEvents`, `ListLoopRouteCauses`, `NextLoopRunEventSequence` — timeline + route-evidence queries.
- `internal/store/globaldb/queries/loop_lifecycle.sql` — node control/attempt/wait lists + the four inventory projections to reuse.
- `internal/store/globaldb/global_db_loop_api.go` + `global_db_loop_events.go` + `global_db_loop_scan.go` — existing read projections + scanning the new reads extend.
- `internal/loop/run_event_kind.go` — the event-kind enum the view map must exhaust.
- `internal/loop/node_inventory.go`, `node_attempt.go`, `node_waits.go`, `retry_due.go`, `wait_due.go` — domain projections feeding roster precedence.
- `internal/loop/coordinator_terminal_helpers.go` — `NodeCellTaskID` (cell_task_id links).
- `internal/api/contract/loop_runs.go:96-258` + `loop_nodes.go:71-83` — payload families the new responses join.
- `internal/api/core/loops.go:319-381` — SSE handler (unchanged; the accelerator + `after_sequence` resume).
- `internal/api/core/loop_interfaces.go` — `LoopAPIService` interface gaining the read methods.
- `internal/api/httpapi/loops_routes.go:35-80` + `internal/api/udsapi/loops_routes.go:38-89` — route registration (parity).
- `internal/daemon/loop_api_runs.go:105,147,384` — `ListLoopRuns`/`GetLoopRun`/`ListLoopRunEvents` service impls the extension lands beside.
- `internal/cli/loop.go:65-87`, `loop_runs.go`, `loop_runs_output.go`, `loop_nodes.go:11-44`, `loop_client.go` — CLI verb homes (no `why` verb exists today).
- `internal/api/spec/loops.go` + `loop_run_event_schemas.go` — OpenAPI authoring for the new routes.
- `skills/compozy/` — official skill loops reference.

### Dependent Files

- `web/src/generated/compozy-openapi.d.ts` — regenerates (consumed by task_05).
- `web/src/systems/loops/adapters/loops-runs-api.ts` + `mocks/*` — task_05 binds to the new reads (no web edits here).
- `web/e2e/fixtures/scenario-contracts.ts` + `runtime-seed.ts` — E2E contract fixtures updated here (co-ship).
- `internal/daemon/loop_api_payloads_test.go`, `internal/api/core/loops_test.go`, `internal/cli/loop_test.go` — suites extended.
- `packages/site/content/docs/cli/loop/**` — CLI docs regenerate via `make codegen` (three new verbs).

### Competitor References

- `.resources/smithers/apps/cli/src/monitor-ui/monitorModel.ts:506,685,285` — verdict cascade, event tiers, attempt sentences (briefing + timeline views).
- `.resources/smithers/packages/ui-core/src/runs/{runProgress,runEta,runHealth}.ts` — truthful-metrics types (`ratio: null`): absent, never fabricated.
- `.resources/smithers/packages/gateway-ui/src/runNodeStatus.ts` — latest-execution-wins status merging (current-state truth).
- `.resources/sim/apps/sim/lib/logs/log-views.ts` — graduated agent read models with scan budgets (view tiers).
- `.resources/temporal/common/persistence/sql/sqlplugin/sqlite/query_converter.go:151-205` — keyset pagination SQL (timeline/roster cursors).
- `.resources/temporal/service/history/api/describeworkflow/api.go:206-237` + `service/history/workflow/activity.go:92-215` — describe-vs-list split; stored-vs-derived per-node projection.

### Related ADRs

- [ADR-005: Run reads are computed projections](adrs/adr-005.md) — the design this task implements.
- [ADR-002: One loop run page, two registers](adrs/adr-002.md) — the briefing verdict serves the default register.
- [ADR-003: Run-bound live DAG enters scope](adrs/adr-003.md) — the roster is the DAG's read model.
- [ADR-006: Terminal settlement is part of the transition](adrs/adr-006.md) — terminal briefing outcome/artifacts source.

## Deliverables

- `RunReadService` projections + three routes on HTTP/UDS + runs-list extension
- CLI verbs matching `_dx.md` transcripts, exit codes, and error table verbatim
- Exhaustive event-kind view map (test-enforced) + snapshot-fenced timeline + durable→live seam
- Regenerated OpenAPI/TS + E2E fixtures (co-ship); official skill loops reference updated
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] UT-001..UT-010, UT-049, UT-051 — briefing cascade: tones, blocker ordering, expiry, terminal variants, queued/dormant, unknown-run error, zero-generation boundary, current-state truth, typed outcome/artifacts
- [ ] UT-011, UT-012, UT-013 — view-map exhaustiveness over the CHECK list; fork beat + coalesced `seq=last`; cursor error family (branch-changed, invalid, beyond-head)
- [ ] UT-014..UT-020, UT-048, UT-050 — roster assembly: total mapping, progress derivation, `not_taken` from route evidence only, `pending` split, filter allowlist pinned to `_dx.md`, fan-out rollups, cancel dispositions, generation history, terminal-before-round-1
- [ ] IT-017, IT-018, IT-019, IT-020 — roster completeness + terminal-faithful re-read; fan-out 100 pagination stability; not-taken filtering; served-verdict parity
- [ ] IT-021, IT-022, IT-023 — timeline gap/dup-free under appends + `--after` resume + two-reader convergence; foreign/fork cursor 409 with body; view tier counts match the map
- [ ] IT-027 — node verb via API reflects in roster read ≤ one poll cycle; stale verb → existing deterministic conflict
- [ ] IT-032 — runs-list ordering holds across page boundaries; `attention`/`progress` correct on HTTP/UDS/CLI
- [ ] E2E-001 — `_dx.md` golden path verbatim (start → clean `task list` → `loop why` → approve → terminal → `--include-loop --loop-run` all settled)
- [ ] E2E-003, E2E-004, E2E-005, E2E-006 — follow resume without gaps; executable unblocker string; attempt recovery read; deterministic error journeys (exit codes + messages from `_dx.md`)

## Success Criteria

- Every assigned test case implemented and passing
- `loop why/nodes/events/runs` output matches `_dx.md` transcripts on a real daemon; errors match the `_dx.md` table verbatim (exit codes + bodies)
- A live reader (first page + SSE at `head_seq`) observes no gaps and no duplicates under continuous appends
- `make codegen-check` green; `make gate` green on the task's diff

### Web/Docs Impact

- `web/`: `web/src/generated/compozy-openapi.d.ts` regenerates (nodes/briefing/timeline payloads + runs-list fields); web components/hooks/fixtures belong to task_05 (`web/src/systems/loops/**`) — no web component edits in this task.
- `packages/site`: generated CLI reference gains `loop why`, `loop nodes` extensions, `loop events` (`content/docs/cli/loop/**` via `make codegen`); generated API reference gains the three routes.
- QA impact: new user-visible CLI verbs + routes → add content-addressed `untested` scenarios for "run briefing + roster + timeline agent journeys (`loop why/nodes/events`)"; flag only — the walk runs in the loop's QA phase.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: no new extension points or hooks; `loop.watch_source` provide surface unaffected; Extension Host API method set unaffected (no loop-run read methods exist there); bridge SDK unaffected; MCP sidecars unaffected. Protocol docs/OpenAPI regenerate via codegen.
- Agent manageability: this task IS the agent surface — structured roster/briefing/timeline reads, `-o json`/`-o jsonl`, resume semantics, deterministic errors per `_dx.md`; closed enums (tones, node states) land in generated references; official skill documents the verbs + audit-reason vocabulary.
- Config lifecycle: none — no `config.toml` keys (checked `internal/config`; pagination limits ride the existing convention).
