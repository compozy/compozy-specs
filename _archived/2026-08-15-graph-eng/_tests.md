# Test Specification: Loop Graph Completion (graph-eng)

Canonical test contract for the graph-eng feature set. Companion to `_spec.md`. Derived from `_user_stories.md` (behavior), `_spec.md` Part II (components), `_dx.md` (CLI/API journeys), and `_uiux.md` (browser journeys). Visual contracts for web E2E and `eng-ui-screenshot` live in `docs/design/opendesign/graph-eng/` (`DESIGN-NOTES.md`).

## Strategy

- Go unit: table-driven, `t.Run("Should …")` + `t.Parallel`, `-race`, fakes only at store/clock/rand boundaries (the coordinator's injected `now`/`retryRand`); canonical suites extended in place (linter tables, coordinator planner suites, namespace tables) per `eng-consolidate-test-suites`.
- Go integration (`+integration`): real SQLite through the migration streams; canonical fresh/reopen/ahead/integrity/equivalence suites extended for every schema change; HTTP=UDS parity via `internal/api/testutil`.
- E2E runtime: `make test-e2e-runtime` (Go harness + acpmock) driving real daemon flows.
- E2E web: `make test-e2e-web` (Playwright + MSW fixtures updated with the new payloads/events).
- Gates: `make gate` per phase, `make gate-full` at close; `make codegen-check` guards contract drift; 80% per-package floor.

## Coverage Matrix

| Source | Behavior | Unit | Integration | E2E |
| --- | --- | --- | --- | --- |
| US-001 | Ask parks run; validated answer becomes output | UT-055, UT-056 | IT-005 | E2E-001 |
| US-001.EC-1 | Answer fails shape → rejected, still pending | UT-056 | — | E2E-020 |
| US-001.EC-2 | Ask without expect → lint error | UT-010 | — | — |
| US-001.EC-3 | Two responders race → one admits | — | IT-002 | — |
| US-001.EC-4 | Answer after expiry → `request_expired` | UT-063 | IT-003 | — |
| US-001.EC-5 | Daemon restart keeps request answerable | — | IT-008 | E2E-012 |
| US-001.EC-6 | Duplicate answer idempotent | UT-064 | IT-010 | — |
| US-001.EC-7 | Per-lane asks answer independently | UT-065 | — | — |
| US-002 | Review pauses; approve/edit executes once | UT-057, UT-058 | IT-006 | E2E-002 |
| US-002.EC-1 | Edited args fail schema → pending intact | UT-058 | — | E2E-021 |
| US-002.EC-2 | Concurrent decisions → one wins | — | IT-002 | — |
| US-002.EC-3 | Reject without note → default feedback | UT-059 | — | — |
| US-002.EC-4 | Review pause doesn't consume action clock | UT-060 | — | — |
| US-003 | Respond-as-result substitutes execution | UT-061 | — | E2E-004 |
| US-003.EC-1 | Response fails output shape | UT-061 | — | — |
| US-003.EC-2 | Respond without output shape → lint | UT-011 | — | — |
| US-003.EC-3 | Decision outside allowlist → rejected | UT-062 | — | — |
| US-004 | Amend parked output with provenance | UT-099, UT-102 | IT-011 | E2E-027 |
| US-004.AC-3 | Amend never re-runs consumers; rerun pairing | UT-099 | IT-011 | — |
| US-004.EC-1 | Amend running node → `amend_not_parked` | UT-100 | — | — |
| US-004.EC-1b | Amend without output → `amend_no_output` | UT-112 | — | — |
| US-004.EC-2 | Amend fails shape → original stands | UT-099 | — | — |
| US-004.EC-3 | Concurrent amends → CAS, one wins | UT-103 | IT-002 | — |
| US-004.EC-4 | No output shape → `amend_schema_missing` | UT-101 | — | — |
| US-004.EC-5 | Amend then cancel → history intact | — | IT-011 | — |
| US-005 | Responder rules: human default, agent opt-in | UT-066, UT-067 | IT-013 | E2E-014 |
| US-005.EC-1 | Spawn-chain child self-denied | UT-068 | — | — |
| US-005.EC-2 | Agent without capability denied first | — | IT-013 | — |
| US-005.EC-3 | Opt-in unanswered → ladder proceeds | UT-069 | — | — |
| US-006 | Ladder: reminder→escalation→expiry route | UT-063 | IT-003 | E2E-012 |
| US-006.EC-1 | Answer vs expiry tiebreak deterministic | UT-063 | IT-003 | — |
| US-006.EC-2 | Unreachable expiry route → lint | UT-014 | — | — |
| US-006.EC-3 | Daemon down across window → fires once | — | IT-003 | E2E-012 |
| US-006.EC-4 | Zero-length ladder durations → lint | UT-014 | — | — |
| US-007 | Requests list shows kind/context/shape | UT-107 | IT-014 | E2E-020 |
| US-007.EC-1 | Context truncation marker | UT-108 | — | — |
| US-007.EC-2 | Deep link after terminal → resolved view | — | — | E2E-022 |
| US-007.EC-3 | Parallel-lane requests listed separately | UT-065 | IT-014 | — |
| US-007.EC-4 | Secret-shaped context redacted | UT-108 | — | — |
| US-008 | Agent lists/answers via structured surfaces | — | IT-004, IT-013 | E2E-014 |
| US-008.EC-1 | Malformed answer → validation vs permission distinct | UT-056, UT-062 | IT-004 | — |
| US-008.EC-2 | Unknown request → not-found | — | IT-004 | — |
| US-008.EC-3 | Re-run answer command idempotent | UT-064 | IT-010 | — |
| US-009 | Router picks one route; default; lint | UT-001..UT-003, UT-020, UT-021 | — | E2E-008 |
| US-009.EC-1 | Two true conditions → declaration order | UT-022 | — | — |
| US-009.EC-2 | Unknown field in condition → lint | UT-004 | — | — |
| US-009.EC-3 | Single route + default valid | UT-005 | — | — |
| US-009.EC-4 | Live-path node still runs | UT-023 | — | — |
| US-009.EC-5 | Broken condition → authoring failure | UT-024, UT-095 | — | — |
| US-010 | Gate verdict selects route | UT-025, UT-026 | — | E2E-008 |
| US-010.EC-1 | Duplicate/conflicting mapping → lint | UT-006 | — | — |
| US-010.EC-2 | Approval-outcome route constraint | UT-027 | — | — |
| US-010.EC-3 | Per-lane routing in fan-out | UT-028 | — | — |
| US-011 | Route cause recorded + skipped ≠ failed | UT-029 | IT-012 | E2E-024 |
| US-011.EC-1 | Default cause says default-after-no-match | UT-021 | — | — |
| US-011.EC-2 | Per-generation decisions independent | UT-029 | — | — |
| US-012 | fail_fast cancels remaining lanes | UT-031, UT-032 | IT-007 | E2E-005 |
| US-012.EC-1 | Simultaneous failures → deterministic trigger | UT-033 | — | — |
| US-012.EC-2 | Failure with all others done → nothing to cancel | UT-034 | — | — |
| US-012.EC-3 | Completed lane never rewritten | UT-035 | IT-007 | — |
| US-012.EC-4 | Restart mid-cancellation completes | — | IT-008 | — |
| US-013 | best_effort partial + coverage declaration | UT-036..UT-038, UT-012 | — | E2E-006 |
| US-013.EC-1 | 100% threshold ≡ wait_all + hint | UT-039 | — | — |
| US-013.EC-2 | Invalid thresholds → lint | UT-013 | — | — |
| US-013.EC-3 | Monotonic admission | UT-040 | — | — |
| US-013.EC-4 | Empty collection semantics | UT-041 | — | — |
| US-013.EC-5 | Partial flag readable downstream | UT-051 | IT-005 | E2E-006 |
| US-013 (run boundary) | `completion_state: partial` at terminal | UT-037b | — | E2E-006 |
| Request cancellation | Atomic with node/run/strategy cancel | UT-111 | IT-015 | — |
| Workspace isolation | Cross-workspace negatives, six verbs + lineage | — | IT-016 | — |
| US-014 | race first-success wins, rest cancel | UT-042 | IT-007 | E2E-007 |
| US-014.EC-1 | Simultaneous successes → one winner | UT-043 | — | — |
| US-014.EC-2 | Race never resurrects losers | UT-044 | — | — |
| US-014.EC-3 | Single-branch race valid | UT-042 | — | — |
| US-015 | progress.* in conditions | UT-050, UT-051 | — | — |
| US-015.EC-1 | Counts settled; totals include unsettled | UT-052 | — | — |
| US-015.EC-2 | Zero-collection rates = 0 | UT-052 | — | — |
| US-015.EC-3 | Sibling-scope reference → lint | UT-053 | — | — |
| US-016 | bind_as/index_as un-shadow nesting | UT-054 | — | E2E-029 |
| US-016.EC-1 | Duplicate names in chain → lint | UT-008 | — | — |
| US-016.EC-2 | Reserved-root collision → lint | UT-008 | — | — |
| US-016.EC-3 | Name outside body → lint | UT-009 | — | — |
| US-017 | Windowed width beyond old cap | UT-045, UT-046 | IT-008 | E2E-013 |
| US-017.EC-1 | Window > count runs all concurrently | UT-046 | — | — |
| US-017.EC-2 | Never-materialized distinct from canceled | UT-047 | — | — |
| US-017.EC-3 | Restart re-forms window, no double-run | — | IT-008 | — |
| US-017.EC-4 | Aggregate display at scale | — | — | E2E-023 |
| US-018 | Absence causes recorded (route/prune/strategy) | UT-029, UT-048 | IT-012 | E2E-024 |
| US-018.EC-1 | Single deterministic cause | UT-048 | — | — |
| US-018.EC-2 | Bounded aggregate prune events | UT-049 | IT-012 | — |
| US-019 | Generation diff kinds | UT-083, UT-084 | — | E2E-025 |
| US-019.EC-1 | Self-diff empty | UT-085 | — | — |
| US-019.EC-2 | Carried marked carried | UT-084 | — | — |
| US-019.EC-3 | Large outputs summarized | UT-086 | — | — |
| US-019.EC-4 | Same-run shape identity (n/a — pinned) | UT-085 | — | — |
| US-020 | Run diff inputs + outcomes + lineage links | UT-087 | IT-004 | E2E-025 |
| US-020.EC-1 | Definition divergence banner | UT-088 | — | — |
| US-020.EC-2 | Cross-loop → `diff_cross_loop` | UT-089 | IT-004 | — |
| US-020.EC-3 | Live-side labeled as-of | UT-087 | — | — |
| US-021 | Rerun opens operator_rerun generation | UT-070..UT-072 | IT-010 | E2E-010 |
| US-021.EC-1 | Parked target rejected | UT-073 | — | — |
| US-021.EC-2 | Mid-generation → `rerun_busy` | UT-074 | — | — |
| US-021.EC-3 | Lane rerun carries siblings | UT-075 | — | — |
| US-021.EC-4 | Idempotent double request | UT-076 | IT-010 | — |
| US-021.EC-5 | Terminal run rerun reactivates | UT-077 | — | E2E-010 |
| US-022 | Fork seeds outputs, overrides inputs, links | UT-078..UT-080 | IT-009 | E2E-011 |
| US-022.EC-1 | Missing blob → deterministic rejection | UT-081 | — | — |
| US-022.EC-2 | Fork of fork chains lineage | UT-080 | — | — |
| US-022.EC-3 | Fork while source live | — | IT-009 | — |
| US-022.EC-4 | No-override replay valid | UT-082 | — | — |
| US-022.EC-5 | Output editing not offered | UT-079 | — | — |
| US-022.EC-6 | Concurrency policy respected | — | IT-009 | — |
| US-023 | Agent time travel under capability rules | — | IT-013 | E2E-014 |
| US-023.EC-1 | Own terminal run allowed | UT-072 | IT-013 | — |
| US-023.EC-2 | Fork of another's live run allowed | — | IT-013 | — |
| US-023.EC-3 | Idempotent under retries | UT-076 | IT-010 | — |
| US-024 | Broken stop_when exits with diagnostic | UT-094 | — | E2E-009 |
| US-024.EC-1 | Data-dependent break exits that generation | UT-094 | — | — |
| US-024.EC-2 | hash_fields present → lint unknown_parameter | UT-015 | — | — |
| US-024.EC-3 | Cost-limit exhaustion + warning surfaced | UT-097, UT-098 | — | — |
| US-025 | Broken routing condition → routable failure | UT-095 | — | — |
| US-025.EC-1 | In-lane failure isolates to lane | UT-096 | — | — |
| US-025.EC-2 | Router broken ≠ default | UT-024 | — | — |
| US-025.EC-3 | on_eval_error override honored | UT-096 | — | — |
| US-026 | Per-gate revision budgets | UT-090, UT-091 | — | — |
| US-026.EC-1 | Per-lane counters | UT-092 | — | — |
| US-026.EC-2 | Operator origins don't consume | UT-093 | — | — |
| Linter (Part II) | All new codes | UT-001..UT-015 | — | — |
| Route planner | Selection/skip/causes | UT-020..UT-029 | — | — |
| Join settlement | Strategies + monotonicity | UT-030..UT-044 | — | — |
| Window engine | Exactly-once lanes | UT-045..UT-049 | IT-008 | E2E-013 |
| Progress projection | Namespace + scopes | UT-050..UT-054 | — | — |
| Request admission | Validation/policy/tiebreaks | UT-055..UT-069, UT-111 | IT-002, IT-003, IT-015 | — |
| Rerun planner | Sets/guards/idempotency | UT-070..UT-077 | IT-010 | — |
| Fork service | Seed/validate/lineage/start transition | UT-078..UT-082, UT-082b | IT-009 | — |
| Blob durability roots | New refs survive the orphan sweep | — | IT-017 | — |
| Diff service | Kinds/summaries/guards | UT-083..UT-089 | — | — |
| Revision counters | Isolation | UT-090..UT-093 | — | — |
| Predicate policy | Defaults/overrides/cost | UT-094..UT-098 | — | — |
| Amend | Guards/provenance/CAS | UT-099..UT-103, UT-112 | IT-011 | — |
| Config lifecycle | Keys/ceilings | UT-104..UT-106 | — | — |
| Contracts/enums/events | Parity + validity + codecs + lineage policy | UT-107..UT-109, UT-113, UT-114 | IT-012 | — |
| Migration (schema) | New table/columns/CHECKs | — | IT-001 | — |
| Transport parity | Six verbs HTTP=UDS | — | IT-004 | — |
| Capabilities | Gates + self-denial | UT-066..UT-068 | IT-013 | E2E-014 |
| `_uiux.md` S1–S11 | Browser flows against `docs/design/opendesign/graph-eng/` | — | — | E2E-020..E2E-030 |
| `_uiux.md` S12 | Editor chrome (`graph-eng-editor-chrome.html`) | — | — | E2E-031..E2E-034 |

## Unit Tests

### Linter (Spec: Implementation Design / lint codes)

- **UT-001** (error): route node without `default` → `route_default_missing` naming the node.
- **UT-002** (error): `routes[].to` targeting a non-forward node (backward and unknown) → `route_target_invalid` per offending route.
- **UT-003** (happy): valid router with 2 routes + default compiles; destinations recorded as dependencies.
- **UT-004** (error): `routes[].when` referencing `nodes.missing.output.x` → `unknown_reference` with available nodes listed.
- **UT-005** (boundary): router with a single route + default compiles.
- **UT-006** (error): `on_result` with duplicate outcome keys or `{route:}` + string action for the same outcome → deterministic lint error.
- **UT-007** (error): `on_result: { fail: branch }` (removed string action) → `route_action_removed` pointing at `{ route: ... }`.
- **UT-008** (error): duplicate `bind_as` in one nesting chain → `iteration_name_conflict`; `bind_as: previous` (reserved root) → same code.
- **UT-009** (error): body-scoped name referenced outside the body → scope error naming the fan-out.
- **UT-010** (error): `ask` without `expect` → lint error requiring the answer shape.
- **UT-011** (error): `review.decisions` containing `respond` on a node with no declared output shape → lint error.
- **UT-012** (error): `strategy: {kind: best_effort, threshold: 66%}` without `missing: acceptable` → `strategy_coverage_undeclared`.
- **UT-013** (boundary): thresholds `0%`, `101%`, `count: 0`, negative count → `strategy_threshold_invalid`; `1%` and `count: 1` compile.
- **UT-014** (error): `expires.route` pointing at a non-forward destination → lint; `expires.after: "0s"` → `duration_invalid`.
- **UT-015** (error): `no_progress.hash_fields: [x]` → `unknown_parameter` (field deleted); fixture sweep proves no fixture still authors it.

### Route planner (Spec: Router)

- **UT-020** (happy): condition true on route 2 only → route 2 continues; routes 1/3 dominated subgraphs skipped with `route_not_taken`.
- **UT-021** (happy): no condition matches → default route; cause records `default`.
- **UT-022** (ordering): routes 1 and 2 both true → route 1 (declaration order) and cause names the matched condition.
- **UT-023** (state): node reachable from selected route and a skipped route → runs via the live path (existing skip rules).
- **UT-024** (error): broken CEL in `routes[].when` → node fails `authoring`/`predicate_evaluation_failed`; default NOT taken.
- **UT-025** (happy): gate outcome `fail: {route: remediate}` → remediate path selected, others skipped, verdict records the route.
- **UT-026** (happy): string actions (`revise`, `continue`) still act unchanged next to object-form routes.
- **UT-027** (error): approval outcome mapped to a route that would bypass approval constraints → planner rejects per legality table.
- **UT-028** (state): gate inside a fan-out lane routes only its lane (`item_index` scoped skip).
- **UT-029** (happy): `route_taken` event payload carries `{route, cause}`; independent per generation.

### Join settlement (Spec: Strategy engine)

- **UT-030** (happy): `wait_all` (default and explicit) reproduces today's barrier bit-for-bit over a fixture matrix.
- **UT-031** (happy): fail_fast — lane 2 definitive failure with lanes 1/3 running → settlement `failed`, cancel intents for 1/3, trigger recorded as lane 2.
- **UT-032** (state): fail_fast with lane failure still retry-eligible → no settlement (strategies act on definitive outcomes only).
- **UT-033** (concurrency): two definitive failures in one plan tick → deterministic single trigger (lowest item index), both recorded failed.
- **UT-034** (boundary): failure arrives after all siblings succeeded → settlement failed, zero cancel intents.
- **UT-035** (state): cancel intent racing a completed lane → completed cell untouched (epoch fence), settlement unchanged.
- **UT-036** (happy): best_effort `threshold: 66%` with 2/3 succeeded, 1 pending → partial admit, pending lane canceled, coverage `{total:3, succeeded:2, coverage_rate:0.67}`.
- **UT-037** (happy): collect cell status `partial` + structured output payload exactly matching `_dx.md`.
- **UT-037b** (happy): a partial join on the terminal path sets `loop_runs.completion_state = 'partial'` at terminal commit; a fully-satisfied run stays `complete`; the projection lands in run list/detail payloads and the SSE `status_changed` payload.
- **UT-038** (error): threshold unmet after all settle → failure path with coverage recorded.
- **UT-039** (boundary): `threshold: 100%` behaves as wait_all; compile hint emitted (warning severity).
- **UT-040** (idempotency): late lane failure after partial admission → settlement unchanged, `late_arrival` event only.
- **UT-041** (boundary): empty collection → existing empty semantics; strategy recorded irrelevant.
- **UT-042** (happy): race — first success wins, output = winner's, others canceled-by-strategy; single-lane race degenerates to plain.
- **UT-043** (concurrency): two successes in one tick → deterministic winner (lowest item index); loser's result retained in history.
- **UT-044** (state): race with all lanes failed → failure path listing all failures.

### Window engine (Spec: Window engine)

- **UT-045** (happy): 500-item collection, `max_parallel: 8` → exactly 8 lanes materialized; lane terminal materializes the next index once.
- **UT-046** (boundary): window ≥ lane count → all materialize immediately; no batching artifacts.
- **UT-047** (state): fail_fast at lane 3 of 500 → unmaterialized lanes settle canceled with never-started cause and zero task runs.
- **UT-048** (happy): absence causes single-valued and deterministic when route-skip and strategy-cancel overlap (first applicable wins).
- **UT-049** (boundary): `branch_pruned` aggregate payload stays under the event byte bound at width 500.

### Progress projection (Spec: Progress projection)

- **UT-050** (happy): `nodes.shard.progress.{total,succeeded,failed,canceled,running,pending,settled,success_rate,failure_rate}` computed from a fixture cell matrix.
- **UT-051** (happy): downstream gate reads `nodes.collect.status == "partial"` and `progress.success_rate` via qualified form.
- **UT-052** (boundary): totals include unsettled lanes; zero-lane collection → all counts 0, rates 0 (no division error).
- **UT-053** (error/happy): bare `progress.*` outside its fan-out's body → lint scope error; qualified `nodes.<fanout>.progress.*` compiles anywhere the node id is referenceable (downstream condition, contract `stop_when`, another fan-out's body).
- **UT-054** (happy): nested fan-outs with `bind_as: file` / `bind_as: line` resolve outer/inner correctly in templates and CEL.

### Request admission (Spec: Request plane)

- **UT-055** (happy): ask opens with rendered prompt/context frozen; parked cell carries wait kind `request`; inline fields are redacted bounded previews and the full redacted context lands behind `context_ref`.
- **UT-056** (error): answer failing the matching per-decision schema (`answer_schema_json` for ask, `edit_schema_json` for edit, `respond_schema_json` for respond) → `request_validation_failed` with field details; request stays pending; state unchanged.
- **UT-111** (state): node cancel / run kill / route skip / strategy cancel of a pending request cell → the request row transitions to `canceled` in the same transaction, `request_canceled` event appends, `aggregates.pending` drops in the same commit, and a later respond returns `request_canceled`.
- **UT-057** (happy): review admission `approve` → NodeRun released with original params snapshot.
- **UT-058** (happy/error): `edit` with valid params → executes edited, record keeps both versions; invalid edit → rejected, pending intact.
- **UT-059** (happy): `reject` without note → default feedback; with `on_reject.route` → route taken; without → node fails `quality_rejection`.
- **UT-060** (state): action `timeout` clock starts at admission, not at request open.
- **UT-061** (happy/error): `respond` payload validates against the node's output shape; failure leaves pending.
- **UT-062** (error): decision outside the request's allowlist → deterministic rejection listing allowed decisions.
- **UT-063** (ordering): expiry vs answer resolved by store transaction order; expiry outcome `route`/`halt` recorded with cause `wait_expired`.
- **UT-064** (idempotency): duplicate answer → `request_already_answered` carrying the recorded decision; store unchanged.
- **UT-065** (state): per-lane requests in a fan-out open/answer independently; sibling lanes unaffected.
- **UT-066** (happy): default responders deny agent actors before validation.
- **UT-067** (happy): `responders.agents: allow` admits a capability-holding agent; provenance records agent identity.
- **UT-068** (error): initiator-chain self-response (starter and spawned child) → `respond_self_denied` regardless of opt-in.
- **UT-069** (state): opted-in node with no answer follows the ladder identically to human-only nodes.

### Rerun planner (Spec: Time travel)

- **UT-070** (happy): rerun from `shard` → rerun set = node + transitive dependents; others carried; origin `operator_rerun`, parent = latest settled generation.
- **UT-071** (happy): provenance records operator actor + reason.
- **UT-072** (happy): agent rerun of its own terminal run allowed; of its own executing run → `timetravel_self_denied`.
- **UT-073** (error): parked/pending target → `rerun_node_unsettled`.
- **UT-074** (error): generation in flight → `rerun_busy`.
- **UT-075** (state): `--item 2` reruns lane 2 + dependents; sibling lanes carried.
- **UT-076** (idempotency): rerun with an explicit `request_id` twice → one generation and one `loop_timetravel_ops` row, replay returns the prior result; the same `request_id` with different arguments → `timetravel_key_reuse`; two keyless reruns with identical arguments → two acknowledged operations (each its own ledger row), with rapid duplicates absorbed by `rerun_busy` while the first generation is in flight.
- **UT-077** (state): terminal run rerun reactivates through normal reactivation with cause recorded.

### Fork service (Spec: Time travel)

- **UT-078** (happy): fork writes the child's pre-settled seed generation 1 (`origin='fork_seed'`, one output row per source cell, blobs copied by content address, `best_generation=1`), pins the source's executed-definition digest, and plans generation 2 (`origin='initial'`, parent 1) as the first executing generation whose `previous.*`/`best.*` hydrate from the seed via the unmodified same-run history reader.
- **UT-079** (happy): input overrides validated exactly like fresh-start inputs; node-output editing not representable in the input type; overridden inputs never coexist with stale carried outputs (all body nodes are in the generation-2 plan; no seed cell is executable).
- **UT-080** (happy): lineage columns written at creation; `forks[]` projection lists children; fork-of-fork chains; each fork records one `loop_timetravel_ops` row in the same transaction.
- **UT-081** (error): missing baseline blob in the seed → deterministic rejection, no partial run; source run outside the caller workspace → unknown-run rejection before any read.
- **UT-082** (boundary): fork with no overrides → inputs identical to source, recorded as such; generation 2 still executes the full body from the seeded baseline.
- **UT-082b** (state): fork start transition through the canonical path — child inserted at `generation = 1` with seed rows settled in the same transaction, coordinator reserved for generation 2 with intent `(2, 1, initial)`; best columns satisfy the both-or-neither CHECK (finite source score → `best_generation=1`+score; no score → both NULL and only `previous.*` hydrates); a `queue`d fork writes its seed at insert and reserves the coordinator on promotion; boot reconcile treats the child as a normal generation-1 run with a pending generation-2 coordinator; fork-of-fork repeats against the fork's own generations.

### Diff service (Spec: Time travel)

- **UT-083** (happy): generation diff emits `changed/rerun/skipped/carried/verdict` rows matching a golden fixture.
- **UT-084** (state): carried-forward cells marked `carried`, never `changed`.
- **UT-085** (boundary): generation diffed against itself → empty diff.
- **UT-086** (boundary): output beyond the payload bound → size + content-hash summary row.
- **UT-087** (happy): run diff includes input rows, per-node rows at chosen/latest settled generations, terminal row; live side labeled as-of.
- **UT-088** (state): different pinned definition versions → divergence header; only shared nodes compared.
- **UT-089** (error): different loop names → `diff_cross_loop`.

### Gate revision counters (Spec: Data Models — B-5)

- **UT-090** (happy): gate A revising twice leaves gate B's budget untouched.
- **UT-091** (boundary): budget reached → declared exhaustion behavior; record names the exhausted gate.
- **UT-092** (state): counters keyed per `(gate, item_index)` in fan-out lanes.
- **UT-093** (state): `operator_rerun` generations do not consume authored budgets.

### Predicate policy (Spec: bugs B-2/B-3)

- **UT-094** (happy): broken `stop_when` → run exits through completion path with diagnostic `predicate_evaluation_failed`; succession NOT blocked.
- **UT-095** (happy): broken routing predicate (branch/route/filter) → authoring failure, routable via `on_error`.
- **UT-096** (state): `on_eval_error: exit` on a routing node and `stop_when: { expr: "...", on_eval_error: fail }` (contract object form) invert the defaults; the `StopWhenSpec` dual-form codec round-trips string and object and rejects malformed shapes; lane-scoped failure isolates.
- **UT-097** (boundary): CEL cost ≥80% of limit → warning surfaced in diagnostics (not discarded).
- **UT-098** (error): cost limit exceeded → classified evaluation failure with cost diagnostic attached.

### Amend (Spec: Request plane)

- **UT-099** (happy/error): valid amend inserts exactly one `loop_node_amendments` row and mutates **no** `loop_generation_outputs` row; the cell's effective output (namespace read, resume, downstream consumption) resolves to the newest amendment while the recorded output stays byte-identical; invalid payload → no row, effective output unchanged.
- **UT-100** (error): running (non-parked) node → `amend_not_parked`.
- **UT-101** (error): node without declared output shape → `amend_schema_missing`.
- **UT-102** (happy): amendment rows are append-only and immutable (`amendment_seq` monotonic per cell); `original_ref` blob retained and readable after multiple amendments.
- **UT-103** (concurrency): two amends race → the `amendment_seq` insert CAS admits one; second must re-read; sequences never collide.
- **UT-112** (error): parked cell with no settled output → `amend_no_output`.

### Config lifecycle (Spec: Config Lifecycle)

- **UT-104** (happy/error): `requests.expire_after = "72h"` parses; `"0s"`/garbage → path-qualified validation error.
- **UT-105** (state): seed applies only to requests with no authored `expires`; authored wins.
- **UT-106** (boundary): `fan_out_width = 500` validates (ceiling removed); negative rejected.

### Contracts, enums, events (Spec: API Endpoints / Data Models)

- **UT-107** (happy): request payload/`aggregates.pending` builders produce the exact `_dx.md` shapes (previews + `answered_at` provenance; no `*_ref` raw payload in any transport type).
- **UT-108** (boundary): context/prompt/proposed previews bounded + redacted at open; truncation marker set; secret-shaped values in the exact `proposed_ref` snapshot never appear in any API/event/log projection.
- **UT-109** (happy): `operator_rerun` + `fork_seed` in the origin enum + ordered `*Values()` parity; the eight new event kinds (`request_opened`, `request_answered`, `request_expired`, `request_canceled`, `node_amended`, `route_taken`, `branch_pruned`, `run_forked`) valid, unknown kind still rejected.
- **UT-113** (happy/error): `StrategyThreshold` codec round-trips exactly `threshold: 66%` and `threshold: {count: 2}` through YAML/JSON and the bijective editor codec; mixed (`{count: 2, percent: 50}`), unknown keys, and malformed scalars are decode errors.
- **UT-114** (happy/error): `ResponderPolicy` chain evaluation — direct starter denied, transitively spawned child denied, unrelated agent allowed, human operator allowed, cross-workspace actor denied, missing/stale lineage fails closed (denied).

## Integration Tests

### Schema migration (canonical suites)

- **IT-001**: fresh apply / reopen-with-data / ahead / integrity / equivalence over the new migration set (`loop_requests`, `loop_runs` fork columns, origin + status + wait-kind + event-kind CHECK rebuilds) — including a pre-migration DB carrying live runs that reopens with every projection intact.

### Store races and idempotency

- **IT-002**: races owned by real authoritative transitions — two concurrent `Respond` transactions → exactly one admits, loser reads `request_already_answered` with the winner's decision; respond racing expiry and respond racing cancel → single deterministic winner by transaction order; amend racing amend → `amendment_seq` CAS admits one; amend racing resume → resume consumes a consistent effective output (the amendment either fully precedes or fully follows).
- **IT-003**: expiry due-scan + timer racing an answer → exactly one outcome; restart replays neither (shared idempotency key).
- **IT-005**: respond → coordinator wake → namespace exposes `nodes.<ask>.output` to a downstream template (store round-trip).
- **IT-006**: review `edit` admission → task enqueue carries the edited snapshot; executed params equal admitted params.
- **IT-007**: fail_fast settlement → post-commit cancel delivery to running lanes (acpmock session receives cancel); completed lane result intact.
- **IT-008**: daemon restart mid-window and mid-cancellation → boot reconcile re-forms the window; no lane runs twice; pending request still answerable.
- **IT-009**: fork while source runs — snapshot-isolated seed; source rows byte-identical after; loop `concurrency: forbid` blocks a fork start exactly like a fresh start.
- **IT-010**: keyed rerun/fork agent-retry → single effect + prior-result replay; keyless repeat → a second real operation with its own ledger row; respond retry → `request_already_answered` idempotent echo.
- **IT-011**: amend → resume → downstream consumes amended output; the run-detail `amendments[]` projection (HTTP/UDS/CLI/native parity) shows original + amended with actor/reason/sequence; amend-then-cancel keeps both.
- **IT-012**: all eight new event kinds (`request_opened`, `request_answered`, `request_expired`, `request_canceled`, `node_amended`, `route_taken`, `branch_pruned`, `run_forked`) append + SSE replay from `seq=0` + `Last-Event-ID` resume; `route_taken`/`branch_pruned` payload contracts; producer enum, durable append, generated contract, and the named web listener change together.
- **IT-013**: capability matrix — `loops.respond`/`loops.timetravel` grants and denials across HTTP/UDS/native tools (diff ungated on all three); self-denial 403s (`respond_self_denied`, `timetravel_self_denied`) for the direct starter and a spawned child, via the shared `ResponderPolicy`.
- **IT-014**: `GET /loop-requests` filtering (`state=pending`; `state=resolved` = the closed union `answered|expired|canceled`), pinned ordering (pending: `expires_at ASC NULLS LAST, opened_at ASC`; resolved: `resolved_at DESC`) with stable cursors across pages and across the union, and `aggregates.pending` equals the daemon count under concurrent opens/answers/cancels.
- **IT-015**: run cancel with a pending request → wait claim + request `canceled` + `request_canceled` event commit in one transaction; restart replays nothing; the bell count drops; respond afterwards → `request_canceled`.
- **IT-016**: cross-workspace negatives — all seven operations (request detail included), lineage projections (`forked_from`/`forks[]`), diff both-sides resolution, SSE routes, and `aggregates.pending` reject or exclude another workspace's runs/requests; fork with a foreign source run → unknown-run rejection.
- **IT-017**: blob durability roots — the orphan sweep retains every blob referenced by `loop_requests` (context/proposed/answered refs, pending and resolved) and `loop_node_amendments` (original/amended, across multiple amendments and after run cancel/terminal), while true orphans are still reclaimed at the existing transition/terminal boundaries; fresh/reopen preserved.

### Transport parity

- **IT-004**: for each of the seven new operations (requests list, request detail, respond, amend, diff, rerun, fork): HTTP and UDS return byte-identical success and error envelopes for the `_dx.md` fixtures (422/409/410/403/404 cases included); the request-detail read returns the full redacted context that the list truncates.

## End-to-End Tests

### Runtime (Go harness + acpmock)

- **E2E-001** (US-001, US-007): golden path — publish `rollout`-style loop → run → parks at ask → `compozy loop requests` shows it → `loop respond` with valid payload → run resumes → downstream reads the answer → terminal `done`.
- **E2E-002** (US-002): review flow — run parks pre-execution → `respond --decision edit` → tool executes exactly once with edited args → record shows original + edited + actor.
- **E2E-003** (US-002.AC-4): `respond --decision reject --note` → `on_reject.route` path runs; no execution of the reviewed action.
- **E2E-004** (US-003): `respond --decision respond` with an output-shaped payload → node succeeds without executing; downstream consumes it.
- **E2E-005** (US-012): 3-lane fan-out fail_fast — lane failure cancels running siblings (acpmock observes cancels); join takes the failure path.
- **E2E-006** (US-013): best_effort 66% with one failing lane → collect `partial` + coverage output; downstream branch on `status == "partial"` takes the partial path; terminal `loop status` shows `status: done` + `completion_state: partial` (never plain complete).
- **E2E-007** (US-014): race — first success settles; losers canceled.
- **E2E-008** (US-009, US-010): router selects by classifier output; a gate `{route:}` verdict reroutes a revise loop; `loop status` shows route causes.
- **E2E-009** (US-024): definition with a broken `stop_when` → run exits `done`-path with `predicate_evaluation_failed` diagnostic instead of iterating.
- **E2E-010** (US-021): `loop rerun --from-node` on a terminal run → generation with origin `operator_rerun`; carried nodes intact; run completes again.
- **E2E-011** (US-022): `loop fork --generation N --input …` → new run seeded, lineage visible from both `loop status` outputs; source untouched (hash comparison).
- **E2E-012** (US-006): request expiry across a daemon restart → escalation effect emitted once, expiry route taken once.
- **E2E-013** (US-017): 500-item fan-out with `max_parallel: 8` → completes; `loop status` progress counts truthful during and after.
- **E2E-014** (US-008, US-023): agent session using native tools — `compozy__loop_requests` → `compozy__loop_respond` (granted) succeeds; self-response and ungranted timetravel denied with deterministic reasons.

### Web (Playwright + MSW)

Boards: `docs/design/opendesign/graph-eng/`. Screenshot evidence binds to task_08 VC-01..VC-16 and task_09 VC-01..VC-02.

- **E2E-020** (S1): pending ask renders prompt/context/expected shape → invalid submit shows field errors inline → valid submit resolves the card.
- **E2E-021** (S1): review card — proposed args table → edit flow pre-fills editor → decision bar reflects the allowlist only.
- **E2E-022** (S1): request answered elsewhere → revisit shows resolved state, no form; deep link to a terminated run shows outcome view.
- **E2E-023** (S3): strategy progress — partial badge with coverage numbers; canceled-by-strategy distinct from failed; wide-run aggregate counts.
- **E2E-024** (S2): timeline rows for route-taken (matched vs default), pruned lane with cause, amended node with provenance, forked run link.
- **E2E-025** (S7): diff view — generation compare with grouped change kinds; run compare with input block and divergence banner fixture.
- **E2E-026** (S8): fork dialog — generation picker + pre-filled inputs → submit navigates to the new run; validation error path.
- **E2E-027** (S5): amend dialog — schema-validated editor, original read-only, success reconciles from refreshed truth (no optimistic paint).
- **E2E-028** (S5): rerun dialog — rerun-set preview; verb absent on a parked node and during an in-flight generation.
- **E2E-029** (S10): editor — add ask + route + strategy + bind_as via inspector → DSL view round-trips losslessly → lint dock shows `route_default_missing` when default removed.
- **E2E-030** (S9, post-herdr): bell shows the loop-request row (workspace label, jump lands on the request form); badge equals sessions `needs_you` + `aggregates.pending`.
- **E2E-031** (S12): editor chrome — a fresh editor opens with both rails collapsed (full-bleed canvas); the toolbar toggle and `[` expand the palette rail (its search field filters kinds); selecting a node auto-opens the inspector on the Node tab; `]` collapses it; a reload restores the persisted chrome state (`compozy:loops:editor-chrome:v1`).
- **E2E-032** (S12): quick-add — `a` with canvas focus opens the dialog while `a` typed inside an inspector input does not; choosing a kind places the node at the visible viewport center, selects it, opens the inspector, and enables Save layout (`positionsDirty` set on a placed add); pane double-click opens the same dialog; jump-to-node reveals and centers the target.
- **E2E-033** (S12): connection-drop — dragging a connection from a source handle onto empty canvas opens the picker at the drop point; choosing a kind creates the node already wired from the source in one atomic step (edge present; publish raises no orphan/unreachable surprise beyond the linter's normal verdict); Escape dismisses with zero mutations.
- **E2E-034** (S12): node menu + selection — right-click on a node offers Duplicate/Copy/Paste/Delete on a workspace loop and hides every mutating verb on a read-only source; marquee-selecting two nodes and pressing Delete removes both nodes and all their edges (no orphan edges left in the draft); a selected edge deletes via its midpoint ✕ affordance.
