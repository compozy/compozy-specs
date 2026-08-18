# Test Specification: Loops Paper Adoption

Canonical test contract for repair context + re-attempt semantics + ratchet + provenance.
Companion to `_techspec.md`. **No `_user_stories.md` exists** (no PRD — scope decided by grill);
journeys are derived from TechSpec behaviors and flagged `J-N` in the matrix.

## Strategy

- Go: table-driven `t.Run("Should …")` subtests, `t.Parallel` default, `-race`/`CGO_ENABLED=1`;
  fakes only at I/O boundaries (store faked in pure-logic units; real SQLite in store/integration
  suites). Route-decision and metric-comparison units are **pure route-contract tests** (paper §6:
  `state → label`, no LLM).
- Placement: extend canonical suites — gate plane in `internal/loop/gate/*_test.go`, coordinator in
  `internal/loop/*_test.go`, store in the globaldb migration/query suites
  (fresh/reopen/ahead/integrity/equivalence), contract/API in `internal/api/*` suites, CLI in
  `internal/cli` golden-JSON suites. No new standalone suites.
- Execution: `make gate` per lane; `make gate-full` at close; `make codegen-check` for contract
  drift; e2e-runtime via `make test-e2e-runtime` (acpmock); web e2e via `make test-e2e-web`.
- E2E mock + matchers co-ship with the contract change (L-007). `t.Setenv` cases drop
  `t.Parallel` (L-002).

## Coverage Matrix

| Source | Behavior | Unit | Integration | E2E |
| --- | --- | --- | --- | --- |
| DSL/linter (TechSpec: Data Models) | metric grammar + lint rules | UT-001–UT-006, UT-041 | — | — |
| Namespace grammar | `previous`/`best` path validation + CEL | UT-007–UT-010 | — | — |
| History projection | `GenerationHistory` assembly (verdicts map + route causes) | UT-011–UT-014 | IT-001, IT-023 | — |
| Metric comparison | direction/min_delta/non-finite/eligibility | UT-015–UT-020 | — | — |
| Score emission | command/judge/extension contracts | UT-021–UT-026 | IT-002 | — |
| Verdict intents + sanitizer | plan intents, redaction, `loop_gate_verdicts` queries | UT-027, UT-040 | IT-003–IT-006, IT-024 | — |
| Provenance | `loop_generations` rows/origins via plan | UT-028–UT-029 | IT-007–IT-008 | — |
| Succession planner | rerun-root union, both `next_generation` surfaces, seeding | UT-030–UT-035 | IT-009–IT-012 | — |
| Schema migration | fresh/hard-cut reopen/equivalence | — | IT-013–IT-015 | — |
| Contract/API/SSE | DTO mapping + `gate_verdict` payload migration | UT-036 | IT-016–IT-019 | — |
| CLI renderers | status detail vs runs summary | UT-037–UT-038 | — | — |
| Hooks payloads | additive fields | UT-039 | IT-020 | — |
| Web reducer | migrated event consumption | UT-042 | — | — |
| Workspace isolation | cross-workspace unreachability | — | IT-021 | — |
| Observability | coverage-matrix extension | — | IT-022 | — |
| J-1 ratchet climb | improve → best advances → done | — | — | E2E-001 |
| J-2 ratchet regression | regress → seed-from-best restore | — | — | E2E-002 |
| J-3 DoD reject → retry | `next_generation` iterates with context | — | — | E2E-003 |
| J-4 livelock regression | both actions bounded, deterministic terminals | — | — | E2E-004, E2E-007 |
| J-5 exhaustion → best | terminal references best generation | — | — | E2E-005 |
| J-6 web run detail | score/best/origin visible | — | — | E2E-006 |

## Unit Tests

### DSL + linter (TechSpec: Data Models — grammar)

- **UT-001** (happy): parsing `metric: {direction: maximize}` on an `agent-judge` criterion yields
  `MetricSpec{Direction: MetricMaximize, MinDelta: nil}`.
- **UT-002** (error): `metric` on a `human` criterion → blocking lint diagnostic (metric placement
  code), definition rejected by `compiler` with `LintFailedError`.
- **UT-003** (error): two criteria with `metric` in one definition → blocking `CodeMetricSingle`.
- **UT-004** (error): `metric: {min_delta: 0.1}` without `direction` → blocking lint (direction
  required).
- **UT-005** (boundary): `min_delta: 0` parses to explicit zero (strict-improvement semantics
  preserved, not "unset").
- **UT-006** (error): `metric: {direction: sideways}` → parse/lint rejection naming the closed
  enum.

### Namespace grammar (`dsl/refs`)

- **UT-007** (happy): `ValidatePath` accepts `previous.generation`,
  `previous.nodes.build.output`, `previous.verdicts.quality.blocking_issues`, `best.score`,
  `best.nodes.build.output`; fan-out namespace construction selects the current item before these
  paths are resolved, so the public grammar remains index-free.
- **UT-008** (error): `previous.bogus` and `best.verdict` → `CodeUnknownReference`.
- **UT-009** (happy): CEL env compiles `best.score >= 0.9 ? "halt" : "revise"`-shaped conditions
  referencing both new roots.
- **UT-010** (state): linter reference builder and `ValidatePath` accept/reject the identical path
  set (table shared across both — grammar-drift guard).

### History projection (`control_namespace_history.go`)

- **UT-011** (boundary): generation 1 → `GenerationHistory{Previous: nil, Best: nil}` and namespace
  renders empty objects (template interpolation of `previous` yields no keys, no error).
- **UT-012** (happy): generation 3 with faked reader rows → `Previous.Generation == 2`, node
  statuses/outputs mapped by `(node ID, item index)`, `Verdicts` keyed by `(gate ID, item index)`
  carries every recorded gen-2 verdict, `RouteCauses` matches the persisted
  `route_cause_rank` order without collapsing fan-out siblings.
- **UT-013** (happy): with `loop_runs.best_generation = 1`, `Best.Nodes` exposes generation-1
  outputs through item-indexed `BestNodeProjection` values (output field only — no status field
  exists on the type), preserving both scalar and object outputs, and `Best.Score` the stored score.
- **UT-014** (state): DoD verdict (contract gate) appears in `Previous.Verdicts` on the
  `dod_retry` generation keyed by the contract gate ID and item index 0, and that gate instance is
  the sole entry in `RouteCauses`.

### Metric comparison (`gate/metric.go`)

- **UT-015** (happy): maximize, best 0.8, candidate 0.9, min_delta 0 → improvement.
- **UT-016** (happy): minimize, best 120, candidate 100, min_delta 10 → improvement.
- **UT-017** (boundary): candidate == best (either direction) → no improvement (strict).
- **UT-018** (boundary): maximize, best 0.8, candidate 0.85, min_delta 0.1 → no improvement.
- **UT-019** (error): candidate `NaN`/`+Inf` → `VerdictOutcomeInvalidOutput`, comparison never
  runs, best untouched.
- **UT-020** (state): best eligibility (B-004) — matrix over the first scored generation: finite
  score + metric-gate aggregate `approved` → becomes best; finite score + aggregate `rejected`
  (sibling criterion failed) → best stays NULL; non-finite score → `invalid_output`, best stays
  NULL.

### Score emission (gate evaluators)

- **UT-021** (happy): command criterion under metric: stdout `{"score": 0.72}` →
  `CriterionResult.Score == 0.72`.
- **UT-022** (error): command stdout missing `score` under a declared metric →
  `VerdictOutcomeInvalidOutput` with deterministic diagnostic.
- **UT-023** (error): command stdout `{"score": "high"}` → `invalid_output` (type violation).
- **UT-024** (happy): agent-judge rubric response with score field populates `Score`; judge
  attempt numbering unchanged.
- **UT-025** (happy): extension criterion response `score` field populates `Score`
  (`evaluator_extension.go` contract).
- **UT-026** (state): criterion without metric never sets `Score` (nil) — and the deleted
  `Confidence` field no longer exists on `CriterionResult` (compile-level guarantee, asserted by
  the updated judge/extension tests' expected structs).

### Verdict record shape (`gate/verdict_store.go`)

- **UT-027** (happy): `VerdictIntent` projection from an aggregated `Verdict` carries gate ID,
  item index, outcome, score, `route_cause_rank`, sanitized blocking/criteria JSON; human
  `loop_gate_decisions` rows are not produced by machine verdicts.
- **UT-040** (error): sanitizer (B-007) — blocking issues containing a raw `compozy_claim_*`
  token, an OAuth code, and an oversized payload come out redacted and size-bounded; the
  sanitized record is what the intent carries (assert no raw secret substring survives).
- **UT-041** (error): lint — `metric: {direction: maximize, min_delta: -0.1}` and
  `min_delta: NaN`/`Infinity` (YAML `.nan`/`.inf`) are blocking diagnostics (B-004).

### Provenance (`LoopGeneration`)

- **UT-028** (happy): origin mapping table — initial/stop_when/reattempt/gate_revise/
  gate_next_generation/dod_retry/ratchet_restore each produce the matching `GenerationOrigin` and
  `ParentGeneration` (N-1, or best for `ratchet_restore`).
- **UT-029** (error): constructing a `GenerationIntent` with `ParentGeneration >= Generation` →
  invariant error (never persisted); DDL CHECK rejects a hand-inserted violating row (N-001).

### Succession planner (`coordinator_succession.go` + `reattemptRerunSet`)

- **UT-030** (happy): revise with TWO route-causing gates G1 (producers {A, B}) and G2 (producers
  {B, E}), dependents {C}: rerun set == deterministic union {A, B, E, G1, G2, C} regardless of
  evaluation order; unrelated succeeded node {D} carried forward with its `OutputRef` (B-003).
- **UT-031** (state): re-run nodes receive namespace with `previous.*` populated (not blank
  outputs) — asserts the ADR-002 delete target behavior is gone.
- **UT-032** (happy): `RouteNextGeneration` on BOTH surfaces (B-002): in-body gate → fresh
  full-body plan with origin `gate_next_generation`; DoD contract gate → fresh full-body plan
  with origin `dod_retry`; neither returns `Terminal{failed, contract}`; both plans carry the
  verdicts into `previous.verdicts.*`.
- **UT-033** (boundary): DoD retry AND in-body `next_generation` when
  `generation == iteration_cap` → `iterationCapTerminal` (`exhausted`), no new plan.
- **UT-034** (happy): metric-gate rejection WITH baseline: carry-forward refs come from
  `best_generation`'s outputs; origin `ratchet_restore`; `previous.*` still shows the rejected
  generation. WITHOUT baseline (best NULL): carry-forward from last generation, origin
  `gate_revise`, no nil dereference (B-004).
- **UT-035** (state): `ReattemptFailedOnly` + rejection rerun roots preserve `ChildLoopRunID`
  only for carried (non-rerun) nodes — matrix over {carried, rerun-root, dependent}.
- **UT-036** (happy): migrated `gate_verdict` SSE payload (B-006) marshals `{node_id, generation,
  gate_id, item_index, verdict, reason, route, blocking_issues, criteria[]{…, score}, score,
  best_generation}` from the sanitized record; no `confidence` key exists anywhere in the
  payload (criterion-level or summary).

### CLI renderers (`internal/cli`)

- **UT-037** (happy): `loop status -o json` (detail response) includes `best_generation`,
  `best_score`, and per-generation `origin`/`parent_generation`/`verdicts[]`; omits best fields
  (not zero-values) when unset.
- **UT-038** (happy): `loop runs -o jsonl` rows carry run-summary fields only
  (`best_generation`, `best_score` on `LoopRunPayload`) — NO per-generation data on the list
  surface (B-008 DTO mapping) — per the golden fixture.

### Hooks (`internal/hooks`)

- **UT-039** (happy): `loop.gate.post` payload carries `{outcome, score, best_generation}`;
  `loop.generation.pre` carries `{origin, parent_generation}`; matcher behavior unchanged for
  existing fields.

### Web reducer (vitest, canonical suite `web/src/systems/loops/hooks/__tests__/use-loop-stream.test.tsx` + `lib` tests)

- **UT-042** (state): migrated `gate_verdict` event (B-006) — reducer consumes `score` (criterion
  and summary) and `best_generation`; a payload containing the legacy `confidence` key is not
  read (field deleted from types); story rows render score.

## Integration Tests

### Store + coordinator wiring (real SQLite)

- **IT-001**: three persisted generations with verdicts — history projection reads previous
  outputs + verdict and best outputs through the real queries (no fakes).
- **IT-002**: command scorer subprocess (fixture script) → score flows evaluator → verdict →
  `loop_gate_verdicts` row.
- **IT-003**: atomicity through the completion plan (B-001) — verdict rows, best update,
  `loop_generations` row, gate `output_ref`, and the durable event all commit in the single
  `CompleteCoordinatorAndEnqueueNext` transaction; induced failure after EACH new mutation rolls
  back the entire plan (matrix: fail-after-verdict, fail-after-best, fail-after-provenance,
  fail-after-event); a stale-claim coordinator (wrong token) persists nothing.
- **IT-004**: `ListGateVerdicts(run, gen)` returns rows ordered by gate and item index;
  `ListRouteCausingVerdicts` returns only ranked rows in `route_cause_rank` order (never
  `decided_at` order).
- **IT-005**: `loop_gate_decisions` remains human-only — a machine verdict writes zero decision
  rows; a human criterion still writes decision rows exactly as before.
- **IT-006**: best tracking with eligibility (B-004) — approved improvement updates
  `loop_runs.best_generation/best_score` in the plan tx; regression leaves them untouched; a
  finite score on an aggregate-rejected gate leaves best NULL; paired-nullability CHECK rejects a
  hand-written half-set row (N-001).
- **IT-007**: every generation-creating path (initial via loop-start tx, stop_when, reattempt,
  gate_revise, gate_next_generation, dod_retry, ratchet_restore) writes exactly one
  `loop_generations` row — `COUNT(*) == loop_runs.generation` equivalence after a mixed-path run.
- **IT-008**: generation payload assembly uses the `loop_generations` query — a run whose events
  were partially pruned still yields complete generation metadata (the counted 1..N loop is gone).
- **IT-009**: livelock regression (red-before/green-after documented in the fix commit), BOTH
  route actions (B-002): `revise` re-runs the producer union with repair context; in-body
  `next_generation` builds a fresh full-body plan (origin `gate_next_generation`); neither path
  re-evaluates byte-identical inputs; revision counter still bounds total revise rounds; exact
  generation counts asserted.
- **IT-010**: DoD reject → new generation whose namespace exposes the DoD verdict in
  `previous.verdicts.*`; second pass approves → run `done`; `loop_generations` shows origin
  `dod_retry`.
- **IT-023**: two parallel gates, including fan-out siblings, rejecting in ONE generation (B-003)
  — durable history retains verdicts keyed by `(gate ID, item index)`, each runtime namespace
  exposes only its scoped gate verdicts, `route_causes` order is stable across repeated runs, and
  the rerun root set is the deterministic union of all gate instances' producers.
- **IT-024**: secret-redaction regression (B-007) — a command criterion emitting a raw
  `compozy_claim_*` token in its blocking issue: assert the raw token is absent from the
  `loop_gate_verdicts` row (SQL), HTTP and UDS payloads, CLI `-o json` output, native-tool
  output, SSE replay, `previous.*` namespace rendering, and captured logs.
- **IT-011**: ratchet restore — gen 2 regresses vs gen 1: gen 3 seeds from gen 1 outputs
  (`OutputRef` equality against gen-1 rows), `parent_generation == 1`, blobs of gen 2 remain
  readable (CAS untouched).
- **IT-012 (ablation, paper §4.3)**: identical definition minus the `metric` block — gen 2's
  regressed output is accepted, best columns stay NULL, origin never `ratchet_restore`; proves the
  metric block is the load-bearing difference, and boolean-gate behavior is byte-compatible with
  pre-spec semantics except ADR-002 fixes.
- **IT-013**: migration fresh-apply — new tables/columns exist with CHECK constraints enforced
  (bad `origin` rejected).
- **IT-014**: migration hard cut (B-005) — a database seeded with pre-change loop runs reopens
  cleanly with ALL pre-change run history deleted (runs, generation outputs, gate decisions, run
  events, goal checkpoints/turns, binding rows, orphaned blobs) while loop definitions, catalog
  entries, and input defaults survive; `COUNT(loop_generations) == loop_runs.generation` holds
  for every run created after reopen.
- **IT-015**: declarative-vs-migrated equivalence + `atlas.sum` integrity suites extended and
  green (`make codegen-check`).
- **IT-016**: HTTP `GET /workspaces/:ws/loop-runs/:id` — generations carry `parent_generation`,
  `origin`, `verdicts[]`; status 200 body asserted field-by-field.
- **IT-017**: SSE `/loop-runs/:id/events` — `gate_verdict` event arrives only after the verdict
  row is durable (replay from `after_seq` includes it); payload matches UT-036 shape.
- **IT-018**: UDS parity — same run yields identical loop-run payload through the UDS route
  (shared `BaseHandlers`).
- **IT-019**: native tool `compozy__loop_status` output validates against its refreshed schema
  digest (descriptor test extended, not duplicated).
- **IT-020**: hook dispatch — a subscribed extension observes `loop.gate.post` with score fields
  on a real gate evaluation; pre-hook patch attempting to mutate verdict fields is rejected as
  unknown patch field.
- **IT-021**: workspace isolation — verdict/generation queries for a run in workspace A return
  zero rows when scoped to workspace B; HTTP route in B → 404 with deterministic error body.
- **IT-022**: observability coverage matrix — build fails if `gate_verdict` or enriched
  `generation_started` events are missing from any new lifecycle path.

## End-to-End Tests

### Runtime harness (`make test-e2e-runtime`, acpmock)

- **E2E-001** (J-1): definition with judge metric (maximize): mock scores 0.5 → 0.7 → 0.9 with
  `stop_when best.score >= 0.9` → run `done`; `best_generation == 3`; CLI `loop status -o json`
  shows the score.
- **E2E-002** (J-2): mock scores 0.8 → 0.6: gen 3 origin `ratchet_restore`,
  `parent_generation == 1`, node inputs seeded from gen-1 outputs; final run outputs == gen-1
  outputs.
- **E2E-003** (J-3): DoD gate rejects pass 1 with blocking issues; mock's pass-2 prompt contains
  the `previous.verdicts.contract.blocking_issues` interpolation (acpmock matcher asserts prompt content);
  pass 2 approves → `done`.
- **E2E-004** (J-4): revise-routing gate with producers succeeded; acpmock fixes the output on
  revision 2 → run ends `done` in EXACTLY 3 generations (deterministic oracle, B-002), and the
  event stream shows producer re-runs each revision; guards the historical livelock.
- **E2E-005** (J-5): metric run hitting `iteration_cap` without reaching target → `exhausted`
  terminal payload references `best_generation`; `loop status` CLI shows best, not last.
- **E2E-007** (J-4): same definition as E2E-004 but acpmock never fixes the output → run ends
  `exhausted` at EXACTLY `max_revisions` revise rounds (deterministic bound assertion).

### Web (`make test-e2e-web`, Playwright — no `force: true`)

- **E2E-006** (J-6): run-detail page for a seeded ratchet run renders score value, best badge on
  the best generation, origin chip "restored from gen 1" on the restore generation, and the
  exhausted outcome card linking best outputs; assertions on visible text + accessible roles, not
  screenshots.
