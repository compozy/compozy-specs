# TechSpec — Loops Paper Adoption (repair context, re-attempt semantics, ratchet, provenance)

## Executive Summary

This spec adopts the validated mechanisms from arXiv 2607.19297 ("Graph-Based Agentic AI with
LangGraph") and the Drive synthesis note into the Loop domain, and records what is rejected or
deferred. No PRD exists; scope was decided in an interactive grill on 2026-08-01 backed by five
exploration passes over current code plus a full paper re-read.

Four coupled changes, all inside the existing loop plane: (1) **cross-generation repair context** —
two new read-only namespace roots, `previous.*` and `best.*`, carrying the prior generation's
outputs and rejecting verdict across the generation boundary (ADR-001); (2) **re-attempt semantics
fix** — gate rejection re-runs the gate's producers with repair context instead of livelocking, and
the DoD `next_generation` route starts a generation instead of terminating the run (ADR-002);
(3) **metric-gated ratchet** — opt-in scored gate criteria with direction, regression-rejects
acceptance, best-so-far tracking, seed-from-best, and queryable machine verdicts (ADR-003);
(4) **lineage-lite provenance** — a `loop_generations` side table recording each generation's
parent and origin (ADR-004). The trade-off accepted throughout: keep the strict-DAG body and the
single linear succession cursor, and express all iteration power at the generation boundary — the
paper re-read proved this is functionally equivalent to its statically-declared cycles once the
failure payload crosses the boundary.

**MVP Boundary**: everything in this spec is one MVP delivered by one task tree — schema migration,
DSL metric block, score emission, verdict persistence, namespace roots, re-attempt semantics,
provenance, contract/CLI/web/docs co-ship, and the trailing qa-report + qa-execution pair.
Post-MVP (explicitly out of scope): multi-metric ratchets, lineage Level A (true tree), parallel
candidate generations (rejected, ADR-004), memory graph plane (deferred —
`.compozy/tasks/_toplan/memory-graph-plane/_briefing.md`), and the loop eval harness (deferred; this spec produces
its raw data).

## System Architecture

### Component Overview

All components live in existing packages; no new packages, no daemon wiring changes beyond
constructor options already flowing through `internal/daemon` loop assembly.

- **Generation history projection** (`internal/loop`, new file `control_namespace_history.go`) —
  assembles `previous.*`/`best.*` from `GenerationOutputReader` + the new verdict/generation
  readers; consumed by the namespace builder at its 8 call sites.
- **Metric gate evaluation** (`internal/loop/gate`) — `MetricSpec` on a criterion; evaluators emit
  `Score`; aggregation compares direction-normalized delta against best; verdict persisted.
- **Succession planner** (`internal/loop`) — rerun-root computation for gate rejection, DoD
  fresh-generation plan, seed-from-best carry-forward, `loop_generations` row writes.
- **Stores** (`internal/store/globaldb`) — new tables `loop_gate_verdicts`, `loop_generations`;
  new columns `loop_runs.best_generation`, `loop_runs.best_score`; sqlc queries.
- **Surfaces** — contract payloads, HTTP/UDS routes (existing paths, extended payloads), CLI
  renderers, native tool schemas, SSE event payloads + one new event kind, web loop system, site
  docs.

Data flow (one generation): coordinator assembles namespace (now including history projection) →
nodes run → gates evaluate (score if metric, diagnostics sanitized) → coordinator builds the
completion plan carrying verdict/best/provenance/event intents → store finalizer applies the plan
inside the token-fenced `CompleteCoordinatorAndEnqueueNext` transaction → SSE broadcast → on
`revise`/`next_generation`: succession planner picks seed source (last vs best) and the
deterministic producer-root union across route-causing gates → next generation.

### Architectural Boundaries

- `internal/loop` owns all behavior; `internal/loop/gate` owns verdict semantics;
  `internal/loop/dsl` owns grammar; `internal/loop/goal` is untouched (its turn-level feedback loop
  remains the intra-node analogue).
- `internal/store/globaldb` owns schema + queries; `internal/loop` consumes via the reader/writer
  interfaces below — no SQL outside the store package.
- `internal/api/contract` → `internal/api/core` → `httpapi`/`udsapi` co-ship payload changes; CLI
  consumes contract types only.
- No package imports `daemon/`; new store readers are injected through existing loop service
  constructor options at the composition root.
- Import graph unchanged — no `magefiles/boundaries.go` update needed (no new package).
- 500-line cap plan (files at/near cap are extended by NEW files, never grown):
  `control_plan.go` (494), `linter_references.go` (493), `linter.go` (457),
  `coordinator_generation.go` (443) are frozen. New files: `control_namespace_history.go`,
  `coordinator_succession.go` (DoD retry + seed selection + rerun roots),
  `dsl/gate_metric.go`, `gate/verdict_store.go`, `gate/metric.go` (comparison),
  `linter_metric.go`, `dsl/refs/namespace_history.go` (path validation for new roots).

## Implementation Design

### Core Interfaces

```go
// internal/loop/gate — metric declaration and scoring
type MetricDirection string

const (
    MetricMaximize MetricDirection = "maximize"
    MetricMinimize MetricDirection = "minimize"
)

type MetricSpec struct {
    Direction MetricDirection `yaml:"direction"`
    MinDelta  *float64        `yaml:"min_delta,omitempty"` // nil => 0: any strict improvement
}

// CriterionResult gains Score; Confidence is deleted (write-only today).
type CriterionResult struct {
    // ...existing fields minus Confidence...
    Score *float64 // set only by metric criteria; non-finite => invalid_output
}
```

```go
// internal/loop/gate/verdict_intent.go — typed mutation intents. There is NO standalone writer
// interface: intents ride GenerationSnapshotPayload / CoordinatorCompletionPlan and are applied
// by the store finalizer inside CompleteCoordinatorAndEnqueueNext (BEGIN IMMEDIATE, claim-token
// fenced). Generation-1 rows are written by the loop-start transaction (B-001).
type VerdictIntent struct {
    GateID         string
    ItemIndex      int             // fan-out materialization identity; 0 outside fan-out
    Outcome        VerdictOutcome
    Score          *float64        // finite; validated before intent construction
    RouteCauseRank *int            // nil = not route-causing; 0..n = deterministic causal order
    BlockingIssues json.RawMessage // SANITIZED diagnostics (Safety Invariant 11)
    Criteria       json.RawMessage // SANITIZED criterion results
}

type BestUpdateIntent struct {
    Generation int64
    Score      float64
}

// Read side stays interface-shaped (reads are not mutations).
type VerdictReader interface {
    ListGateVerdicts(ctx context.Context, workspaceID, runID string, generation int64) ([]VerdictRecord, error)
    ListRouteCausingVerdicts(ctx context.Context, workspaceID, runID string, generation int64) ([]VerdictRecord, error)
}
```

```go
// internal/loop — generation provenance (lineage-lite, ADR-004). GenerationIntent rides the
// coordinator completion plan (or the loop-start transaction for generation 1) — no standalone
// recorder interface exists (B-001).
type GenerationOrigin string

const (
    OriginInitial            GenerationOrigin = "initial"
    OriginStopWhen           GenerationOrigin = "stop_when"
    OriginReattempt          GenerationOrigin = "reattempt"
    OriginGateRevise         GenerationOrigin = "gate_revise"
    OriginGateNextGeneration GenerationOrigin = "gate_next_generation"
    OriginDoDRetry           GenerationOrigin = "dod_retry"
    OriginRatchetRestore     GenerationOrigin = "ratchet_restore"
)

type GenerationIntent struct {
    Generation       int64
    ParentGeneration int64 // 0 for initial; best generation on ratchet_restore
    Origin           GenerationOrigin
}

type GenerationLineageReader interface {
    ListGenerations(ctx context.Context, workspaceID, runID string) ([]LoopGeneration, error)
}
```

```go
// internal/loop/control_namespace_history.go — namespace projection (read-only). Verdicts are
// LOSSLESS: keyed by gate ID and item index (a generation can evaluate several gates and fan-out
// materializations), with the deterministic route-causing order persisted as route_cause_rank
// (B-003). Runtime namespaces select the current fan-out item before exposing the gate-ID path.
type GenerationHistory struct {
    Previous *PreviousGeneration // nil on generation 1
    Best     *BestGeneration     // nil until an eligible score records a best
}

type PreviousGeneration struct {
    Generation  int64
    Nodes       map[string]map[int]NodeProjection    // node ID -> item index -> status + output
    Verdicts    map[string]map[int]VerdictProjection // gate ID -> item index -> verdict
    RouteCauses []GateInstanceProjection             // route-causing gate instances, in rank order
}

type GateInstanceProjection struct {
    GateID    string
    ItemIndex int
}

type BestNodeProjection struct {
    Output any // output only; scalar and object values are both preserved (N-002)
}

type BestGeneration struct {
    Generation int64
    Score      float64
    Nodes      map[string]map[int]BestNodeProjection
}
```

Error handling follows package conventions: store errors wrapped `%w`; non-finite or missing score
under a declared metric maps to the existing `VerdictOutcomeInvalidOutput`; all new paths emit
canonical loop events with correlation keys.

### Data Models

New schema (declarative source: `internal/store/globaldb/schema/definitions/50_loops.sql`; next
gap-free Goose migration appended; `atlas.sum` + sqlc regenerated via `make codegen`).

```sql
-- one row per generation; workspace scope derives from loop_runs FK
-- (same pattern as loop_generation_outputs). Invariants enforced in DDL (N-001).
CREATE TABLE loop_generations (
    loop_run_id       TEXT    NOT NULL REFERENCES loop_runs(id) ON DELETE CASCADE,
    generation        INTEGER NOT NULL CHECK (generation >= 1),   -- monotonic +1, never reused
    parent_generation INTEGER NOT NULL DEFAULT 0
        CHECK (parent_generation >= 0 AND parent_generation < generation),
    origin            TEXT    NOT NULL CHECK (origin IN
        ('initial','stop_when','reattempt','gate_revise','gate_next_generation',
         'dod_retry','ratchet_restore')),
    created_at        TIMESTAMP NOT NULL,
    PRIMARY KEY (loop_run_id, generation)
);

-- machine verdicts, gate-grained (human approvals stay in loop_gate_decisions).
-- route_cause_rank persists the deterministic causal order of route-deciding gates (B-003).
CREATE TABLE loop_gate_verdicts (
    loop_run_id          TEXT    NOT NULL REFERENCES loop_runs(id) ON DELETE CASCADE,
    generation           INTEGER NOT NULL CHECK (generation >= 1),
    gate_id              TEXT    NOT NULL,
    item_index            INTEGER NOT NULL DEFAULT 0 CHECK (item_index >= 0),
    outcome              TEXT    NOT NULL CHECK (outcome IN
        ('approved','rejected','awaiting_approval','blocked','error','timeout','invalid_output')),
    score                REAL,               -- NULL unless metric criterion; finiteness Go-enforced
    route_cause_rank     INTEGER,            -- NULL = not route-causing; 0..n = causal order
    blocking_issues_json TEXT    NOT NULL DEFAULT '[]' CHECK (json_valid(blocking_issues_json)),
    criteria_json        TEXT    NOT NULL DEFAULT '[]' CHECK (json_valid(criteria_json)),
    decided_at           TIMESTAMP NOT NULL,
    PRIMARY KEY (loop_run_id, generation, gate_id, item_index)
);

-- best tracking on the run row (declarative source adds the paired-nullability table CHECK;
-- Atlas generates the table rewrite — acceptable because pre-change run history is hard-cut)
-- loop_runs.best_generation INTEGER  -- NULL until first ELIGIBLE scored acceptance (B-004)
-- loop_runs.best_score      REAL     -- direction-normalized comparisons happen in Go
-- CHECK ((best_generation IS NULL) = (best_score IS NULL))
```

Field rationale:

- `loop_generations.parent_generation` — derivation pointer for audit/UI and the schema seed for a
  future lineage Level A; written once, immutable.
- `loop_generations.origin` — closed enum: why this generation exists; replaces guesswork in UI
  timeline and event payloads.
- `loop_gate_verdicts.outcome`, `.score`, `.gate_id`, `.item_index`, `.generation` — matchable
  columns. `(gate_id, item_index)` is the identity of a materialized gate verdict, including
  fan-out siblings (queries: route-causing verdicts, verdicts per generation, score history).
- `loop_gate_verdicts.route_cause_rank` — deterministic causal order of the gates whose verdicts
  decided the generation's route; `decided_at` is never a tie-breaker (B-003). Written as part of
  the completion plan, immutable afterwards.
- `loop_gate_verdicts.blocking_issues_json`, `.criteria_json` — opaque diagnostic payloads; never
  filtered on; consumed whole by namespace projection and API.
- `loop_runs.best_generation` / `best_score` — the ratchet baseline; single-valued because v1 is
  single-metric (lint-enforced).

Side-table-vs-JSON decisions: machine verdicts are a **side table** (matchable state: outcome,
score, gate, item index, generation — queried by succession planner, API, and future eval spec); generation
provenance is a **side table** (queried to build payloads and lineage); blocking issues and
criterion breakdowns are **JSON columns** inside the verdict row (opaque diagnostics, read whole);
best tracking is **columns on `loop_runs`** (single-valued run state, no new entity). The gate
node's CAS `output_ref` remains the namespace materialization of the same verdict — single writer,
same transaction, verdict table is the queryable projection (no dual authority).

DSL grammar (`compozy.loop/v1`): `GateCriterion` gains optional `metric: {direction, min_delta}`.
Lint rules (new, blocking): `metric` valid only on `command|agent-judge|extension` criteria (not
`human`); at most one metric criterion per definition; `direction` required when `metric` present;
`min_delta`, when present, must be finite and non-negative (B-004).
Namespace grammar: `previous` and `best` roots added to `ValidatePath`
(`dsl/refs/namespace.go:77-109`), the CEL variable set (`dsl/refs/condition.go:55-60`), and linter
reference builders — same paths valid in templates and CEL. Verdict paths are map-shaped:
`previous.verdicts.<gate_id>.{outcome,score,blocking_issues,criteria}` plus
`previous.route_causes` (ordered gate-ID list); there is no singular `previous.verdict` root.
The durable projection retains `(node_id|gate_id, item_index)`, while namespace construction
selects the current fan-out item before rendering these paths, so sibling items cannot collide or
leak and the public path grammar does not gain an index segment.

### API Endpoints

No new routes. Existing payloads extended (contract → OpenAPI → TS via `make codegen`), with one
exact DTO mapping so list/status/native summaries agree (B-008):

- `LoopRunPayload` (summary — used by list endpoints, `loop runs`, `loop list` latest-run,
  `compozy__loop_runs`): `+ best_generation?, best_score?`. Summaries never embed per-generation
  history.
- `LoopRunResponse` (detail — used by `loop status`, run page, `compozy__loop_status`):
  `Generations[]` gains `parent_generation, origin, verdicts[] {gate_id, outcome, score,
  item_index, route_cause_rank?}`; payload assembly switches from the counted `1..run.Generation` loop
  (`internal/daemon/loop_api_runs.go:303-321`) to a `loop_generations` query.
- SSE (`/loop-runs/:run_id/events`): `generation_started` payload gains `{origin,
  parent_generation}`. **`gate_verdict` is an EXISTING event kind and this is a payload
  migration, not an addition (B-006).** Before: `{node_id, generation, verdict, reason, route,
  blocking_issues, criteria[]{…, confidence}, confidence}` (summary confidence computed in
  `global_db_loop_event_boundaries.go:133-146`). After (hard replacement, same kind): `{node_id,
  generation, gate_id, item_index, verdict, reason, route, blocking_issues, criteria[]{…, score},
  score, best_generation}` — `confidence` keys deleted (see Delete Targets), payloads built from
  the sanitized verdict record, emitted only after the completion-plan transaction commits
  (durable-append-then-broadcast preserved). All consumers move in the same change: store builder
  + tests, `web/src/systems/loops/lib/loop-events.ts` reducer, generated TS enums/types, story
  rows, component tests, mocks, docs, official skill.
- UDS mirrors HTTP unchanged (shared `BaseHandlers`).

## Impact Analysis

| Component | Impact Type | Description and Risk | Required Action |
| --- | --- | --- | --- |
| `internal/loop` coordinator/succession | modified | New seed selection, rerun roots, DoD retry plan; medium risk (core loop semantics) | New `coordinator_succession.go`; regression matrix in `_tests.md` |
| `internal/loop/gate` | modified | Score emission, metric comparison, verdict persistence; medium | New files `metric.go`, `verdict_store.go`; delete `Confidence` |
| `internal/loop/dsl` + linter | modified | `metric` block, two namespace roots, 3 lint rules; low | New `gate_metric.go`, `linter_metric.go`, `refs/namespace_history.go` |
| `internal/store/globaldb` | modified | 2 tables + 2 columns + queries; finalizer applies plan intents; migration hard-cuts pre-change run history; medium | Goose migration + sqlc + Atlas via `make codegen`; extend rollback suite |
| `internal/api/contract` + codegen | modified | Payload/enum additions; low | `make codegen`; TS types regen |
| `internal/cli` loop renderers | modified | best/score/origin columns in `status`/`runs`; low | Renderer updates + golden JSON assertions |
| Native tools (`compozy__loop_status/runs`) | modified | Output schema fields; low | Descriptor/schema digests refresh in `internal/tools/builtin` + `toolmeta` |
| Hooks (`loop.gate.post`, `loop.generation.pre`) | modified | Additive payload fields (score/outcome; origin/parent); low | `internal/hooks` payload structs + matcher docs |
| `web/src/systems/loops` | modified | Run detail: score/best badge, provenance chips, exhausted→best pointer; `gate_verdict` payload migration (confidence→score) in reducer/stories/tests; medium | Components + generated types + `loop-events.ts` reducer; Storybook stories |
| `packages/site` loop docs | modified | Grammar, ratchet page, evidence-gating section; low | See Web/Docs Impact |
| `internal/loop/goal` | none | Turn-level feedback loop untouched — checked `goal/route.go`, `goal/templates.go`; namespace roots are generation-scoped only | — |

## Extensibility Integration Plan

- **Hooks**: `loop.gate.post` payload gains `{outcome, score, best_generation}`; `loop.generation.pre`
  gains `{origin, parent_generation}`. Additive; matchers unchanged
  (`internal/hooks/events.go:141-147` family intact). Pre-hook `LoopControlPatch` cannot mutate
  verdict/best state (observe-only fields).
- **Extensions as scorers**: the `extension` criterion type's response contract gains the optional
  `score` field — an extension can now be the metric source for a ratchet gate
  (`internal/loop/gate/evaluator_extension.go`). Documented in the extension protocol docs.
- **Criterion-kind registry**: stays a closed enum — no new extension surface for registering
  criterion kinds (unchanged posture, re-checked `dsl/gate_start.go:21-30`).
- **Skills/capabilities, bundles, registries, bridge SDKs, MCP sidecars**: no impact — loop
  execution never crosses those surfaces (checked: no loop resource kind exists in
  `internal/bundles`/`internal/registry`/`internal/marketplace`; ADR-001 keeps execution off the
  network).
- **Official Compozy skill** (`skills/compozy/`): update the loops section — `previous.*`/`best.*`
  roots, metric criteria, re-attempt semantics, provenance fields.

## Agent Manageability Plan

- CLI (mirrors the B-008 DTO mapping): `compozy loop runs`/`loop list` render run-summary fields
  (`best_generation`, `best_score`) only; `compozy loop status` renders the detail response with
  per-generation `origin`, `parent_generation`, and verdict outcome/score. Both in table and
  `-o json|jsonl`; deterministic errors unchanged. `compozy loop inspect` shows metric criterion
  declarations. No new verbs.
- HTTP/UDS: extended payloads above give agents full read parity; verdict history is retrievable
  per run via the existing run/generation endpoints (no new route needed — verdicts ride
  `LoopGenerationPayload.verdicts[]`).
- Native tools: `compozy__loop_status`/`compozy__loop_runs` schemas carry the new fields — agents
  operating loops see scores/best/provenance without the web UI.
- Discoverability: `compozy loop validate` surfaces the new lint codes (metric placement,
  single-metric, unknown namespace path) with deterministic diagnostics.

## Config Lifecycle

**No `config.toml` keys added, changed, or removed.** Ratchet and repair context are DSL-declared
per definition (metric block, template references); repair semantics are unconditional behavior
fixes. Checked surfaces: `internal/config/loops.go` (`LoopsConfigKey`), `defaults.go:85`,
`merge_loops.go`, `tool_surface_loops.go:11-36` (agent-mutable keys) — existing bounds
(`gates.max_revisions`, `iteration_cap`, budgets) already govern the new paths and their defaults
are unchanged. No docs/examples/validation churn.

## Web/Docs Impact

- `web/src/systems/loops/`: `types.ts` (regen), run detail components
  (`components/detail/loop-detail.tsx` score/best), `components/runs/*` timeline (origin chip,
  "seeded from gen N", verdict outcome/score per generation), `loop-run-outcome-card.tsx`
  (exhausted/stalled point at best generation). Storybook stories for new states; screenshot
  verification via `eng-ui-screenshot` before completion.
- `packages/site/content/docs/loops/`: `dsl-reference.mdx` (+`metric` block),
  `reference-grammar.mdx` (+`previous`/`best` roots), `guardrails.mdx` (+re-attempt semantics,
  +evidence-gating pattern section — "weak evidence is a route, not a prompt instruction"), new
  `ratchet.mdx` (concept + example), `running.mdx` (origin/best in run inspection),
  `api/loops.mdx` + CLI docs pages regenerate.
- **QA impact**: add `untested` content-addressed scenarios — ratchet loop end-to-end (improve →
  best advances; regress → seed-from-best), DoD reject → new generation, revise-with-context walk;
  reset `qa_status` of any existing gate-routing scenario touched by the semantics change. Walk all
  flagged scenarios per `qa-execution` before completion.

## Safety Invariants

1. At most one metric criterion per loop definition (blocking lint `CodeMetricSingle`);
   `min_delta` is finite and non-negative (blocking lint).
2. Best eligibility (B-004): `best_generation`/`best_score` advance only when the score is finite
   AND the metric gate's aggregate verdict is `approved` AND the improvement over the current best
   is direction-aware strict ≥ `min_delta`. A rejected/blocked/invalid gate never advances best,
   including the first score.
3. No-baseline branch (B-004): a metric-gate rejection while `best_generation IS NULL` seeds the
   next generation from the last generation (origin `gate_revise`/`gate_next_generation`, never
   `ratchet_restore`); best is never fabricated and never nil-dereferenced.
4. Every verdict, best-update, provenance, and event mutation rides typed intents on
   `GenerationSnapshotPayload`/`CoordinatorCompletionPlan` and is applied by the store finalizer
   inside `CompleteCoordinatorAndEnqueueNext` (`BEGIN IMMEDIATE`, claim-token fenced). No
   standalone writer path exists; a stale or lost coordinator cannot persist any of them (B-001,
   L-005). Generation-1 rows are written by the loop-start transaction.
5. Prior generations' rows and blobs are immutable — repair/ratchet seeding only appends new
   generation rows referencing existing `output_ref`s (CAS untouched).
6. `loop_generations.parent_generation < generation`, written once in the transaction that creates
   the generation; every generation-creating path writes exactly one row (equivalence:
   `COUNT(loop_generations) == loop_runs.generation` per run, valid from first post-migration
   open — see Delete Targets).
7. Verdict rows + gate node `output_ref` are projections of one sanitized record written by the
   single finalizer transaction — no second authority (`loop_gate_decisions` remains human-only).
8. Route determinism (B-003): every route-causing gate instance is identified by
   `(gate_id, item_index)` and its order is persisted as `route_cause_rank` in the same completion
   plan; producer-rerun roots are the deterministic union of ancestors across ALL route-causing
   gate instances; `decided_at` is never an ordering input.
9. Generation numbering stays monotonic `+1`, never reused; lineage is provenance-only (no
   branching, one live leaf) — stall/breaker/cap arithmetic semantics unchanged.
10. Repair paths cannot bypass bounds: `iteration_cap`, budgets, `max_revisions`, stall detection
    and circuit breaker apply to gate-revise, gate-next-generation, DoD-retry, and ratchet-restore
    generations identically.
11. Sanitization boundary (B-007): evaluator-produced diagnostics (`blocking_issues`, criterion
    results, judge/extension/command output fragments) are normalized, secret-redacted
    (claim-token redactor + existing diagnostics redaction/bounding primitives), and size-bounded
    ONCE before intent construction; the verdict row, gate `output_ref`, SSE payload, API/CLI/
    native-tool projections, `previous.*` namespace, and logs all derive from that sanitized
    record. Raw `compozy_claim_*` or other secrets never persist or cross any surface.
12. `previous.*`/`best.*` are read-only projections; no template/CEL path can mutate history.
13. Non-finite or contract-violating scores under a declared metric yield
    `VerdictOutcomeInvalidOutput` — never a silent pass, never a best update.
14. SSE `gate_verdict` and `generation_started` broadcast only after the completion-plan
    transaction commits (append-then-broadcast preserved).

## Delete Targets

No fallback, no compat shim, no placeholder anywhere in this spec: old behaviors are deleted in the
same change that replaces them.

- `terminalFromDefinitionOfDoneVerdict` `RouteNextGeneration → Terminal{failed, contract}` branch
  (`internal/loop/control_contract_gate.go:127-133`) — replaced by the DoD fresh-generation plan.
- Blank-output revise seeding in `reattemptPendingOutput`
  (`internal/loop/coordinator_generation_reattempt.go:240-245` behavior) — replaced by
  repair-context/seed-from-best carry rules.
- `CriterionResult.Confidence` (`internal/loop/gate/types.go:159`), its writers
  (`gate/judge.go:142,180`, `evaluator_extension.go:90`), **and its existing consumers** (B-006 —
  the field is read today, not write-only): the `confidence` criterion + summary payload keys in
  `internal/store/globaldb/global_db_loop_event_boundaries.go` (`:18,:109-110,:133-146`), the web
  reducer's confidence handling in `web/src/systems/loops/lib/loop-events.ts`, story rows,
  component tests, mocks, generated TS types, and any docs mentioning gate confidence — all
  replaced by `score` under the metric contract in the same change.
- Pre-change loop-run HISTORY (B-005, greenfield hard cut): the appended Goose migration deletes
  existing `loop_runs` rows and their children (`loop_generation_outputs`, `loop_gate_decisions`,
  `loop_run_events`, `loop_goal_checkpoints`, `loop_goal_turns`, session-binding/cleanup rows,
  orphaned `loop_output_blobs`) atomically. Loop DEFINITIONS, catalog entries, and input defaults
  are preserved. Rationale: historical generations cannot be truthfully backfilled with
  provenance/verdict rows, and preserving them would permanently violate Safety Invariant 6;
  alpha run history is disposable state (L-006/L-008 allowed exception, documented here).
- Counted generation-payload loop in `internal/daemon/loop_api_runs.go:303-321` — replaced by the
  `loop_generations` query.

## Testing Approach

Strategy (all concrete cases in `_tests.md`): unit tests own grammar/lint, metric comparison,
rerun-root computation, namespace projection, and route decisions as **pure route-contract tests**
(paper §6 doctrine — `state → label`, no LLM); store tests extend the canonical
fresh/reopen/ahead/integrity/equivalence migration suites; integration tests drive the coordinator
succession matrix against a real SQLite store; e2e-runtime (Go harness + `acpmock`) walks ratchet
improve/regress/exhaust, DoD reject → retry, and the livelock regression (red-before/green-after,
deterministic terminal assertions — never `done|exhausted` alternatives); **ablation cases**
(paper §4.3 technique) prove each mechanism is load-bearing by removing it and asserting the
degraded outcome. The finalizer suite induces a failure after every new mutation in the completion
plan plus a stale-claim case (B-001), and secret-redaction regressions assert claim-token absence
across SQL, HTTP, UDS, CLI, native tools, SSE replay, `previous.*`, and logs (B-007). Frameworks/gates: `make gate` per lane during the loop,
`make gate-full` at close; contract changes co-ship `make codegen-check`; web via Turbo from repo
root; E2E mock + matchers ship with the contract change (L-007).

## Development Sequencing

### Build Order

1. Schema: `50_loops.sql` fragments + Goose migration + sqlc/Atlas (`make codegen`) — no behavior.
2. DSL + linter: `metric` block, namespace-root validation, 3 lint rules (parse/lint tests green).
3. Gate plane: score emission (command/judge/extension), metric comparison, verdict persistence,
   best tracking.
4. Namespace history projection + 8 call-site threading + CEL vars.
5. Succession: rerun roots, DoD retry plan, seed-from-best, `loop_generations` writes; delete
   targets removed here.
6. Events/contract/API/CLI/native-tool schemas + `make codegen`.
7. Web loop system + stories + screenshot evidence.
8. Site docs (grammar, ratchet, evidence-gating, regenerated references).
9. QA tail: qa-report planning + qa-execution walks (scenarios above).

### Technical Dependencies

None external. Internal: step 5 depends on 3–4; steps 6–8 depend on 5. Schema first (step 1) so
every later step tests against the final shape.

## Monitoring and Observability

- Canonical loop events extended: `generation_started{origin, parent_generation}`, existing
  `gate_verdict{gate_id, item_index, outcome, score, best_generation}` — both
  durable-append-then-broadcast.
- `slog` fields on new paths: `loop_run_id`, `generation`, `gate_id`, `origin`,
  `parent_generation`, `score`, `best_generation`.
- Coverage-matrix test extended so every new lifecycle path fails the build if its canonical event
  is missing.

## Technical Considerations

### Key Decisions

Recorded as ADRs (below). Summary: generation-boundary repair over in-body cycles (ADR-001);
producer-scoped rejection re-runs + DoD retry (ADR-002); acceptance-coupled single-metric ratchet
with seed-from-best and a dedicated verdict side table (ADR-003); provenance-only lineage, Level B
rejected, Level A deferred (ADR-004). Deferred with pointers: memory graph plane
(`.compozy/tasks/_toplan/memory-graph-plane/_briefing.md`), loop eval harness (consumes this spec's verdict data).

### Known Risks

- **Prompt payload growth** from `previous.*` interpolation — mitigated by documenting field-level
  access; existing blob budgets bound worst case.
- **Behavioral shift** for definitions that relied on DoD rejection terminating the run — greenfield
  accepted; `halt` route remains the explicit stop; release notes call it out.
- **Producer-scoped re-runs on dense graphs** approach `full_body` cost — documented guidance;
  bounds unchanged.
- **Score-source quality** (judge rubric drift) — deterministic `invalid_output` on contract
  violation; ratchet never advances on invalid scores.

## Architecture Decision Records

- [ADR-001: Generation-boundary repair context instead of in-body cycles](adrs/adr-001.md) —
  `previous.*`/`best.*` roots; strict DAG retained (G1 adopt, G3 reject).
- [ADR-002: Gate-rejection re-attempt semantics](adrs/adr-002.md) — producer rerun-roots with
  repair context; DoD `next_generation` starts a generation (G2 fix + delete targets).
- [ADR-003: Metric-gated ratchet](adrs/adr-003.md) — scored criteria, regression-rejects,
  seed-from-best, `loop_gate_verdicts` (H1).
- [ADR-004: Generation provenance instead of a lineage DAG](adrs/adr-004.md) — `loop_generations`
  lineage-lite; Level B rejected; Level A deferred (H2).

## Compozy Impact Audit

- **Native tools**: `compozy__loop_status`, `compozy__loop_runs` output schemas gain
  best/score/origin/verdict fields → descriptor + schema digest refresh in
  `internal/tools/builtin/loops.go` + `internal/toolmeta/native_entries.go`; no new tool IDs; no
  capability-gate changes (`ToolIDLoopApprove` gating untouched).
- **Extensibility and hooks**: additive hook payload fields (`loop.gate.post`,
  `loop.generation.pre`); `extension` criterion response contract gains `score`; criterion-kind
  enum stays closed; no manifest/bundle/registry/bridge/MCP changes (checked — no loop resource
  kind exists); config lifecycle: zero key changes (evidence in Config Lifecycle).
- **Workspace data isolation**: new rows are loop-run-scoped via `loop_run_id` FK CASCADE, same
  pattern as `loop_generation_outputs`; list/read paths go through existing workspace-scoped run
  queries; SSE events carry `workspace_id` as today; no cross-workspace read path added (tests
  assert verdict/generation queries are unreachable across workspaces).
- **Official Compozy skill**: `skills/compozy/` loops section updated (new roots, metric criteria,
  re-attempt semantics, provenance fields) in the docs step.

## Assumptions / Defaults

- No PRD exists; this TechSpec is the scope authority (grill of 2026-08-01).
- `min_delta` defaults to 0 (any strict improvement); absence of `metric` on every gate keeps a
  definition's behavior identical to today except the G2 semantics fixes, which are unconditional.
- `previous.*` refers to generation `N-1` by number (the rejected generation on ratchet restores —
  diagnosis), while carry-forward seeding follows origin rules (best on `ratchet_restore`).
- Single-metric-per-definition is a v1 simplification; multi-objective is future work and rejected
  from this scope.
- Drive synthesis note (doc 2) was unavailable for re-read (MCP disconnected); its digest in
  `.claude/ledger/2026-07-27-MEMORY-paper-loops-gap.md` is the working source, and every H-gap was
  re-validated against code regardless.
- Conversation in BR-PT; all artifacts (this spec, ADRs, tests, code, docs) in English (SD-003).
