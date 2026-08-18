# Test Specification: Loop Node Lifecycle & Failure Contract

Canonical test contract for the node lifecycle & failure contract. Companion to `_techspec.md`.
Derived from `_user_stories.md` (behavior) and `_techspec.md` (components). IDs are permanent once
tasks reference them; dropped cases are marked `(withdrawn)` in place.

## Strategy

- Frameworks and harnesses: Go `testing` with `t.Run("Should …")` subtests + `t.Parallel`
  (`eng-test-conventions`); `-race`/`CGO_ENABLED=1`; fakes only at I/O boundaries (session
  status, tool caller, deadentity store, clockwork clocks); real SQLite for every store and
  coordinator integration case; `acpmock` for e2e-runtime; Playwright for e2e-web.
- Execution: unit via `make test`; integration via `make test-integration` (`+integration` tag,
  co-located); daemon journeys via `make test-e2e-runtime`; browser via `make test-e2e-web`; all
  through `make gate` / `make gate-full`.
- Conventions: table-driven truth tables for classifier/lint/routing; deterministic terminal
  assertions (never `done|exhausted` alternatives); status-code AND body assertions on every API
  case; injected clocks — no `time.Sleep` orchestration; secret-redaction asserted with planted
  `compozy_claim_*` markers.

## Coverage Matrix

| Source | Behavior | Unit | Integration | E2E |
| --- | --- | --- | --- | --- |
| US-001 | Payload-declared failure fails the node | UT-001..UT-004 | IT-001 | E2E-001 |
| US-001.EC-1 | error field wins over success indicator | UT-005 | — | — |
| US-001.EC-2 | empty/null error passes; empty containers fail | UT-006 | — | — |
| US-001.EC-3 | non-inspectable body skips detection + diagnostic | UT-007 | — | — |
| US-001.EC-4 | invalid result_contract is an authoring error | UT-008 | — | — |
| US-001.EC-5 | oversized message truncated with marker | UT-009 | — | — |
| US-002 | Authoring vs runtime distinction | UT-010..UT-012 | IT-002 | — |
| US-002.EC-1 | ref to quarantined/canceled node resolves last output/absent | UT-013 | — | — |
| US-002.EC-2 | deep path names deepest valid segment | UT-014 | — | — |
| US-002.EC-3 | stale node name suggests existing nodes | UT-015 | — | — |
| US-002.EC-4 | mid-run authoring failure never auto-retries | UT-016 | — | — |
| US-003 | Hints flow into repair context and retry feedback | UT-017, UT-018 | IT-003 | — |
| US-003.EC-1 | hint sanitized (secrets/size) | UT-019 | — | — |
| US-003.EC-2 | absorbed failure records hint, injects nowhere | UT-020 | — | — |
| US-003.EC-3 | conflicting hints kept per node | UT-021 | — | — |
| US-004 | Predicate policy: continuation open, routing closed, recorded | UT-022..UT-025 | IT-004 | — |
| US-004.EC-1 | 80% cost warn; >100% broken-predicate policy | UT-026 | — | — |
| US-004.EC-2 | per-predicate policy override honored | UT-027 | — | — |
| US-004.EC-3 | history roots on gen 1 resolve absent, not broken | UT-028 | — | — |
| US-005 | Error route activates, handled, mutually exclusive | UT-029..UT-032 | IT-005 | E2E-002 |
| US-005.EC-1 | route target failure follows normal precedence | UT-033 | IT-005 | — |
| US-005.EC-2 | dead route on infallible node warns | UT-034 | — | — |
| US-005.EC-3 | success leaves route skipped-absent | UT-035 | — | — |
| US-005.EC-4 | cancel preempts route, on_cancel fires | UT-036 | IT-016 | — |
| US-005.EC-5 | parallel lanes route independently | — | IT-006 | — |
| US-006 | allow_fail absorbs with record, absent output | UT-037, UT-038 | IT-007 | — |
| US-006.EC-1 | repeated absorption never trips breaker | UT-039 | — | — |
| US-006.EC-2 | allow_fail + route rejected by lint | UT-040 | — | — |
| US-006.EC-3 | history refs to absorbed output resolve absent | UT-041 | — | — |
| US-007 | Unannotated failure escalates with context | UT-042 | IT-008 | E2E-003 |
| US-007.EC-1 | gen-1 escalation with absent previous | UT-043 | IT-008 | — |
| US-007.EC-2 | quarantined producer → needs-attention | UT-044 | IT-017 | — |
| US-007.EC-3 | pending pause lands at boundary after handling | — | IT-009 | — |
| US-008 | Mechanical auto-retry: backoff, visibility, exhaust | UT-045..UT-050 | IT-010 | E2E-001 |
| US-008.EC-1 | cancel during backoff suppresses retry | UT-051 | IT-016 | — |
| US-008.EC-2 | pause parks pending retry; resume decides reset | UT-052 | IT-018 | — |
| US-008.EC-3 | every attempt counts toward caps/budgets | UT-053 | IT-010 | — |
| US-008.EC-4 | open breaker fails remaining retries fast | UT-054 | IT-022 | — |
| US-008.EC-5 | attempts:0 disables; node wins over defaults | UT-055 | — | — |
| US-008.EC-6 | backoff past deadline → budget_exhausted class | UT-056 | — | — |
| US-009 | Expensive nodes never blind-retry; opt-in continues | UT-057, UT-058 | IT-011 | — |
| US-009.EC-1 | declined checkpoint starts cold, recorded | UT-059 | — | — |
| US-009.EC-2 | large agent retry cap lint-warns | UT-060 | — | — |
| US-009.EC-3 | death ≠ semantic failure; distinct records | UT-061 | IT-024 | — |
| US-010 | Opt-in timeout/deadline classified + routable | UT-062..UT-065 | IT-012 | — |
| US-010.EC-1 | parked states suspend clocks | UT-066 | IT-018 | — |
| US-010.EC-2 | timeout > deadline rejected | UT-067 | — | — |
| US-010.EC-3 | completion/timeout race: one outcome + late diagnostic | UT-068 | IT-012 | — |
| US-010.EC-4 | stale-epoch fire drops with diagnostic | UT-069 | IT-013 | — |
| US-011 | on_error effect once with full context | UT-070..UT-073 | IT-014 | E2E-004 |
| US-011.EC-1 | kill fires no node-trigger effects | UT-074 | IT-025 | — |
| US-011.EC-2 | missing tool → recorded failure; authoring warn | UT-075 | — | — |
| US-011.EC-3 | crash window → at-least-once, stable identity | — | IT-015 | — |
| US-011.EC-4 | absorbed failure still fires on_error | UT-076 | — | — |
| US-012 | Approval link deliverable + decidable | UT-077 | IT-019 | E2E-005 |
| US-012.EC-1 | used link after resolve → already-decided | UT-078 | IT-019 | — |
| US-012.EC-2 | concurrent approvers: single claim | — | IT-020 | — |
| US-012.EC-3 | unauthorized link denied + audited | — | IT-019 | — |
| US-012.EC-4 | dead channel: pause still visible/decidable | UT-079 | — | — |
| US-013 | Terminal-outcome effects fire once per run | UT-080, UT-081 | IT-014 | E2E-003 |
| US-013.EC-1 | crash at terminal → at-least-once | — | IT-015 | — |
| US-013.EC-2 | node + loop scopes both fire, distinguished | UT-082 | — | — |
| US-013.EC-3 | no-op terminal effect fires | UT-083 | — | — |
| US-014 | Events truthful, ordered, deduplicable | UT-084 | IT-015 | — |
| US-014.EC-1 | replay equals live emissions | — | IT-015 | — |
| US-014.EC-2 | per-node ordering under burst | — | IT-015 | — |
| US-014.EC-3 | custom emit kinds same contract | UT-085 | IT-014 | — |
| US-015 | Days-long node, no time-based anything | UT-086, UT-087 | IT-021 | E2E-006 |
| US-015.EC-1 | budgets bound work-time only | UT-088 | IT-018 | — |
| US-015.EC-2 | transport blip ≠ death (degraded signal) | UT-089 | — | — |
| US-015.EC-3 | active-but-not-progressing stays healthy | UT-090 | — | — |
| US-016 | Confirmed death → bounded continuation | UT-091..UT-093 | IT-024 | E2E-006 |
| US-016.EC-1 | death while paused: no resume | UT-094 | — | — |
| US-016.EC-2 | death while awaiting approval: wait unaffected | UT-095 | IT-024 | — |
| US-016.EC-3 | no checkpoint → cold start, still bounded | UT-096 | — | — |
| US-016.EC-4 | death vs cancel race: cancel wins | UT-097 | IT-016 | — |
| US-016.EC-5 | ambiguity → silence path, never resume | UT-098 | — | — |
| US-017 | Silence flags, self-clears, verbs listed | UT-099..UT-101 | IT-021 | — |
| US-017.EC-1 | in-flight tool = life (gate-full case) | UT-102 | IT-021 | — |
| US-017.EC-2 | window 0 disables; death-resume independent | UT-103 | — | — |
| US-017.EC-3 | flag → death confirmed → resume takes over | UT-104 | — | — |
| US-017.EC-4 | flag aggregation per run/workspace | — | IT-026 | — |
| US-018 | Control verbs via liveness channel + kill fallback | UT-105 | IT-016 | E2E-007 |
| US-018.EC-1 | idempotent double-cancel / after-terminal | UT-106 | IT-016 | — |
| US-018.EC-2 | cancel vs completion race reports actual outcome | UT-107 | IT-016 | — |
| US-018.EC-3 | grace exceeded: visible state, no hidden auto-kill | UT-108 | — | — |
| US-019 | Node pause live, provenance, resume variants, parity | UT-109..UT-112 | IT-018 | E2E-008 |
| US-019.EC-1 | pause mid-backoff parks retry | UT-052 | IT-018 | — |
| US-019.EC-2 | idempotent pause / invalid resume answers | UT-113 | — | — |
| US-019.EC-3 | pause on terminal run → invalid-state answer | UT-114 | — | — |
| US-019.EC-4 | all lanes paused → paused-dominant, no stall | UT-115 | IT-018 | — |
| US-019.EC-5 | late async result recorded without waking | UT-116 | — | — |
| US-020 | Auto-pause rules match, provenance, no re-fire | UT-117..UT-119 | IT-023 | — |
| US-020.EC-1 | broad first-attempt match honored + diagnosable | UT-120 | — | — |
| US-020.EC-2 | two rules: deterministic first-match-wins | UT-121 | — | — |
| US-020.EC-3 | rule-paused required producer → needs-attention | UT-044 | IT-017 | — |
| US-021 | Durable waits survive restart; exactly-one resume | UT-122..UT-124 | IT-027 | E2E-009 |
| US-021.EC-1 | fan-out lane wait identity isolation | UT-125 | IT-027 | — |
| US-021.EC-2 | timer due while paused parks the wake | UT-126 | — | — |
| US-021.EC-3 | ahead arrival consumed on entry (default) / rejected | UT-127 | — | — |
| US-021.EC-4 | restart during admission: no double resume | — | IT-027 | — |
| US-022 | Waiting inventory with age; authored ladders fire | UT-128, UT-129 | IT-026 | E2E-010 |
| US-022.EC-1 | ladder effect failure: fail-open, wait visible | UT-130 | — | — |
| US-022.EC-2 | decision mid-ladder wins, steps cancel | UT-131 | IT-027 | — |
| US-022.EC-3 | truthful empty inventory | UT-132 | IT-026 | — |
| US-023 | Per-target breaker: open/fail-fast/probe/close | UT-133..UT-136 | IT-022 | E2E-011 |
| US-023.EC-1 | all targets open → degraded, diagnosable | — | IT-022 | — |
| US-023.EC-2 | shared target health across runs (workspace) | — | IT-022 | — |
| US-023.EC-3 | pending retries fail fast on open target | UT-054 | IT-022 | — |
| US-024 | Quarantine entry complete; requeue via succession | UT-137..UT-140 | IT-017 | E2E-012 |
| US-024.EC-1 | requeue on terminal run → invalid state | UT-141 | — | — |
| US-024.EC-2 | requeue with open breaker fails fast visibly | UT-142 | IT-022 | — |
| US-024.EC-3 | repeat episodes append history, no cap bypass | UT-143 | IT-017 | — |
| US-024.EC-4 | on_quarantine effect with entry context | UT-144 | IT-014 | — |
| US-024.EC-5 | entry sanitized (secrets) | UT-145 | — | — |
| US-025 | One delivered event, one run; loud durable suppression | UT-146..UT-148 | IT-028 | E2E-013 |
| US-025.EC-1 | distinct events both admit (overlap out of scope) | UT-149 | IT-028 | — |
| US-025.EC-2 | duplicate after terminal still suppressed in horizon | UT-150 | IT-028 | — |
| US-025.EC-3 | extension source identity contract enforced | UT-151, UT-192 | — | — |
| US-025.EC-4 | suppression storm countable per source | UT-152 | IT-028 | — |
| US-026 | Verb/state parity, deterministic errors, one truth | UT-153 | IT-029 | E2E-014 |
| US-026.EC-1 | concurrent verb single-claim + winner provenance | UT-154 | IT-020 | — |
| US-026.EC-2 | cross-workspace unchanged (no new crossing) | — | IT-030 | — |
| US-026.EC-3 | pagination, stable order, truncation indicated | UT-155 | IT-026 | — |
| US-027 | Family defaults configurable, validated, resolved | UT-156..UT-160 | IT-031 | — |
| US-027.EC-1 | mid-flight config change: admission-time pin | UT-161 | IT-031 | — |
| US-027.EC-2 | invalid rule rejected; valid rules untouched | UT-162 | — | — |
| US-027.EC-3 | removed defaults → shipped defaults | UT-163 | — | — |
| US-028 | Editor authors Spec 1 failure contract truthfully | WT-005..WT-008 | — | E2E-016 |
| US-028.EC-1 | route XOR allow_fail blocks Publish | WT-006 | — | — |
| US-028.EC-2 | empty on_* folds omit zero-count chrome | WT-007 | — | — |
| US-028.EC-6 | Start-binding allowlist out of Spec 1 scope | — | — | — |
| US-029 | Hero path catalog → run form → detail truthful | WT-009..WT-010 | — | QA walk |
| US-029.EC-1 | catalog empty filters → truthful empty + clear | WT-009 | — | — |
| US-029.EC-2 | required run input blank → Start gated/rejected | WT-010 | — | — |
| US-029.EC-4 | canceled terminal in roster and recent runs | WT-009 | — | — |
| Classifier (TechSpec: Core Interfaces) | class truth table + order | UT-164..UT-169 | — | — |
| DSL/lint (TechSpec: DSL Grammar) | every new lint code | UT-170..UT-176 | — | — |
| Scheduling (ADR-012) | due-scan, stamps, opportunistic timer | UT-177..UT-179 | IT-013 | — |
| Store (ADR-011) | migrations, two-writer race, CASCADE | — | IT-032..IT-034 | — |
| Effects dispatcher (ADR-015) | isolation, identity, cursor | UT-180..UT-182 | IT-014, IT-015 | — |
| Surfaces (TechSpec: API) | routes, CLI, native tools, digests | UT-183..UT-186 | IT-029 | E2E-014 |
| Delete targets (TechSpec) | stop gone; 7m30s kill gone | UT-188 (UT-187 withdrawn) | IT-021 | E2E-007 |
| Sub-loop boundary (ADR-016.6) | parent-close + typed failure | UT-189..UT-191 | IT-035 | — |
| Run cancel/kill terminal (r2 B-001) | canceled outcome + run machine | UT-193 | IT-016 | E2E-007 |
| Effect delivery protocol (r2 B-004) | delivery_key idempotency + boot drain | UT-194 | IT-015 | — |
| Death authority (r2 B-007) | ResumeDeadNode atomicity + cell retirement | UT-195 | IT-024 | — |
| Effect render failure (r3 B-004) | error-as-data delivery | UT-196 | IT-014 | — |
| Expensive-family retry rule (r3 B-005) | DSL-only opt-in, both families | UT-057, UT-197 | IT-011 | — |
| Web run UI (Web/Docs Impact) | reducer kinds, controls, inventories | WT-001..WT-004 | — | E2E-015 |
| Web editor (ADR-018 / US-028) | lifecycle grammar + chrome states | WT-005..WT-008 | — | E2E-016 |
| Web hero path (ADR-018 / US-029) | catalog, run form, detail | WT-009..WT-010 | — | QA walk |

Vitest (web, runs under `make bun-test` via Turbo):

- **WT-001** reducer folds each of the 15 new event kinds without dropping retained frames.
- **WT-002** story rows render retry/pause/quarantine/wait/attention/effect-result rows from
  fixtures.
- **WT-003** progress derivation excludes parked states.
- **WT-004** node control actions gate on daemon-truth state (no verb rendered for states the
  payload doesn't declare).
- **WT-005** (US-028 AC-1): editor reliability envelope + terminal/node `on_*` fields round-trip
  through PATCH/publish mock — after save+reopen, `deadline`, `retry`, `result_contract`,
  `on_error`, and authored effect lists match the persisted definition; no invented keys.
- **WT-006** (US-028 AC-2/EC-1): UI enforces `on_error` route XOR allow_fail and effect one-of
  `emit`/`tool` — setting both absorption modes (or neither effect kind) keeps Publish disabled
  and/or surfaces a named diagnostic; valid single-mode authoring enables Publish when the dock
  has no errors.
- **WT-007** (US-028 AC-4/EC-2): lint dock — fixture with only `wait_expiry_without_path` warning
  leaves Publish enabled and shows the warning; fixture with a blocking error disables Publish;
  fixture with zero issues renders no issue counter/badge.
- **WT-008** (US-028 AC-5/AC-6): chrome states — built-in source fixture renders read-only strip +
  Fork, Publish disabled, Validate available; publish 422 fixture renders publish-rejected danger
  strip with issue list and does not advance the version pill.
- **WT-009** (US-029 AC-1/EC-1/EC-4): catalog renders built-in/custom groups, status filter
  options include `canceled`, empty filter result shows truthful empty + clear-filters; a
  `canceled` loop/run fixture shows the canceled status pill (no `stop` chrome).
- **WT-010** (US-029 AC-2/AC-3/EC-2): run form — required input blank keeps Start disabled or
  rejects submit with a field-named message and creates no run; Ways to start identifies this
  form as `http`; no `stop` control is rendered.

## Unit Tests

### Failure classifier (`failure_classify.go`; ADR-013)

- **UT-164** (happy): `ClassifyNodeFailure` — table over each evidence shape → exactly the 8
  classes; unknown application error → `payload_declared`.
- **UT-165** (ordering): evidence carrying cancel provenance AND payload error → `cancellation`
  (fixed order wins).
- **UT-166** (happy): `ToolError` codes map — `tool_unavailable|tool_backend_failed|tool_timed_out`
  → `transport`; `tool_invalid_input|tool_denied` → `payload_declared`; `tool_canceled` →
  `cancellation`.
- **UT-167** (boundary): lease-expiry sweep evidence → `transport` with target attribution.
- **UT-168** (happy): `RetryAfter` extracted and clamped to node backoff max.
- **UT-169** (state): classifier is pure — same evidence twice → identical result, no side
  effects (run with `-race` under parallel calls).
- **UT-001** (happy): transport-success body `{error: "boom"}` → node failed, cause `boom`,
  class `payload_declared`.
- **UT-002** (happy): `{success: false}` and `{success: "False"}` → failed with
  declared-failure cause naming `success`.
- **UT-003** (happy): declared `result_contract{failure_field: err}` matches → failure per
  contract; built-ins skipped.
- **UT-004** (error): payload failure never marked retry-eligible (US-001 AC-4).
- **UT-005** (ordering): body with `error` + `success: true` → error wins; recorded cause names
  the winner field.
- **UT-006** (boundary): `error: ""` and `error: null` pass; `error: {}` and `error: []` fail
  with "declared error was empty".
- **UT-007** (error): oversized/binary body → detection skipped, `payload_inspection_skipped`
  diagnostic recorded, transport status decides.
- **UT-008** (error): `result_contract` naming a field absent from the node's `Produces` schema →
  `CodeResultContractInvalid` at validation.
- **UT-009** (boundary): cause exceeding the diagnostic bound → truncated with explicit marker,
  never dropped.

### Authoring vs runtime diagnostics (`refs` + classifier)

- **UT-010** (error): reference to missing field on a ran node → authoring-class diagnostic with
  path, node, and available fields list.
- **UT-011** (happy): reference to branch-skipped node → absent value, no error.
- **UT-012** (happy): compile-time detectable bad ref → same diagnostic shape from
  `loop validate` as at runtime.
- **UT-013** (state): refs to quarantined/canceled nodes → last recorded output else absent;
  never authoring error.
- **UT-014** (boundary): deep path wrong at leaf → diagnostic names deepest valid segment +
  fields at that level.
- **UT-015** (happy): stale node name → suggestion list contains existing node ids.
- **UT-016** (error): runtime dynamic authoring failure → class `authoring`, no auto-retry,
  escalates by default.

### Hints (`ActionFailure.Recovery` propagation)

- **UT-017** (happy): failing tool with `recovery` set → hint present in repair context beside
  cause next generation.
- **UT-018** (happy): no hint → generic "review the failure and adjust" guidance injected.
- **UT-019** (error): hint containing `compozy_claim_*` + oversized text → redacted and bounded
  before ledger/context.
- **UT-020** (state): absorbed failure → hint on the attempt row, absent from repair context.
- **UT-021** (state): two failed nodes with different hints → repair context lists per node,
  never merged.

### Predicate policy (`refs.ConditionCompiler` + policy seam)

- **UT-022** (error): `stop_when` throwing at eval → loop exits, exit reason names the predicate.
- **UT-023** (error): branch condition throwing → routable `authoring`-class failure; route
  catches it; unrouted escalates.
- **UT-024** (state): both cases append a predicate diagnostic event (improvement over Sim).
- **UT-025** (error): uncompilable expression → validation failure naming the expression.
- **UT-026** (boundary): cost at 80% → warn diagnostic; over limit → broken-predicate policy of
  the kind.
- **UT-027** (happy): per-predicate `on_eval_error` override flips the default policy.
- **UT-028** (boundary): `previous.*` on generation 1 inside a predicate → absent data, not a
  broken predicate.

### Error routes and absorption (`coordinator_lifecycle.go`)

- **UT-029** (happy): declared route + exhausted retries → route edge activates, success path
  skip-cascades, failure marked handled.
- **UT-030** (state): handled failure → excluded from breaker accounting, included in
  caps/budgets.
- **UT-031** (error): upstream-pointing route → `CodeErrorRouteBackward` lint.
- **UT-032** (happy): failure with route + `on_error.effects` → effects fire AND route taken.
- **UT-033** (state): route target's own unhandled failure → escalates to succession (no route
  loop possible — forward-only).
- **UT-034** (error): route on infallible node → `CodeErrorRouteDead` warning.
- **UT-035** (happy): node succeeded → downstream-of-error refs resolve absent.
- **UT-036** (ordering): cancel in flight preempts route; `on_cancel` fires; close reason =
  canceled.
- **UT-037** (happy): allow_fail → run continues, disposition `absorbed`, downstream refs absent.
- **UT-038** (state): absorption impossible without declaration — unannotated failure never
  absorbs.
- **UT-039** (state): repeated absorbed failures → visible per generation, breaker untouched.
- **UT-040** (error): `route` + `allow_fail` together → `CodeErrorRouteConflict`.
- **UT-041** (happy): `previous.nodes.<absorbed>` resolves absent per namespace contract.
- **UT-042** (happy): unannotated failure (post family retries) → generation ends; succession
  carries classified failure + hint in repair context.
- **UT-043** (boundary): generation-1 escalation → repair context with absent previous.
- **UT-044** (state): rerun set requiring quarantined/rule-paused producer → plan yields
  needs-attention naming the dependency.

### Retry planner (`coordinator_retry.go`; ADR-012)

- **UT-045** (happy): transport failure on mechanical node, no config → `retrying`,
  `next_attempt_at = now + base`, attempt ledger row `retried`.
- **UT-046** (happy): backoff sequence honors decorrelated jitter within `[base, max]` using
  seeded RandFloat64.
- **UT-047** (happy): attempt/next-attempt/last-failure projected into `LoopGenerationOutput`
  payload.
- **UT-048** (happy): `retry_after` from failing attempt overrides curve within bounds.
- **UT-049** (happy): exhausted retries → final failure enters route → effects → escalation
  exactly like a first failure.
- **UT-050** (state): semantic/authoring/cancel classes → zero retries regardless of config
  (per-family table).
- **UT-051** (ordering): cancel during backoff → pending retry epoch-invalidated, no further
  attempts, `on_cancel` fires.
- **UT-052** (state): pause during backoff parks the retry; resume `plain` keeps attempt count,
  `reset_attempts` zeroes it, `immediate` skips remaining delay.
- **UT-053** (state): attempts count toward `iteration_cap`/budget arithmetic inputs.
- **UT-054** (happy): open target breaker → pending retry fails fast `target_unavailable`,
  sequence ends early.
- **UT-055** (boundary): node `max_attempts: 0` → no auto-retry; node declaration beats family
  default.
- **UT-056** (boundary): computed backoff crossing `deadline` → close `budget_exhausted`,
  distinct from last attempt's class.
- **UT-057** (state): table-driven over agent AND sub-loop families — no declaration → no
  retry on transport failure; nonzero family defaults and nonzero `loop_config` retry values
  are IGNORED for both families (config layers can never enable expensive-family retry); only
  the node-level DSL declaration opts in.
- **UT-058** (happy): authored agent retry + existing checkpoint → attempt receives checkpoint
  (continuation input), not cold start.
- **UT-059** (state): checkpoint declined by work → cold start recorded on attempt row.
- **UT-060** (error): agent retry cap above bound → lint warning (unbounded expensive
  re-execution).
- **UT-061** (state): session-death evidence → resume path (not retry); records distinct
  (`resumed` vs `retried`) dispositions.
- **UT-197** (happy): authored `retry:` on a sub-loop node → the retry is scheduled and runs
  (positive opt-in case for the run-loop family, mirroring the agent case in UT-058).

### Timeouts and deadlines

- **UT-062** (happy): no declared limits → no time-based failure at any simulated duration.
- **UT-063** (happy): attempt exceeds `timeout` → class `attempt_timeout`, retry-eligible on
  mechanical, routable.
- **UT-064** (happy): `deadline` exceeded across attempts+waits → `budget_exhausted`, enters
  normal flow.
- **UT-065** (error): invalid duration strings → `CodeDurationInvalid`, never silent default.
- **UT-066** (state): paused/waiting suspends both clocks; anchors shift on resume.
- **UT-067** (error): `timeout > deadline` → `CodeTimeoutExceedsDeadline`.
- **UT-068** (ordering): completion vs timeout race → single outcome; late result appends
  `late_arrival` diagnostic only.
- **UT-069** (state): fired schedule whose issued epoch differs from the cell epoch → dropped +
  `stale_schedule_dropped` diagnostic.

### Scheduling/stamps (`scheduler_loop_due_scan.go` seam)

- **UT-177** (happy): due-scan selects only due, unpaused, live rows; paging cursor stable.
- **UT-178** (state): every superseding lifecycle mutation (pause/cancel/resume/quarantine/
  requeue/route) increments the epoch of every affected cell in the same transaction; scheduled
  work persists its issuing epoch; `loop_node_controls.revision` never gates schedules.
- **UT-179** (happy): opportunistic timer and due-scan produce the same idempotency key — double
  fire deduplicates.

### Effects (`effects_dispatch.go`, `effect_context.go`; ADR-015)

- **UT-070** (happy): terminal node failure → `on_error` effect fires exactly once with class,
  cause, hint, node identity, run link pre-bound.
- **UT-071** (happy): `on_retry` fires per attempt with attempt number + next-attempt time.
- **UT-072** (error): failing effect (dead tool) → recorded in `effect_results`; node outcome
  untouched.
- **UT-073** (state): multiple entries on one trigger → isolated; first failing never stops the
  second; per-entry results.
- **UT-074** (state): node/run kill → no node-trigger (`on_*`) effects dispatched; kill event
  present; contract terminal effects are UT-081's scope.
- **UT-075** (error): unknown effect tool at runtime → deterministic `tool_missing` result;
  authoring-time warning lint.
- **UT-076** (happy): allow_fail + `on_error` → effect fires with absorbed disposition in
  payload.
- **UT-080** (happy): contract terminal effects fire once per run per outcome; subset
  declaration valid (no-op for undeclared).
- **UT-081** (state): operator cancel or kill reaching a terminal outcome → the contract
  terminal effect for that outcome fires exactly once (terminal effects ride terminal truth,
  kill included; node-trigger effects stay suppressed on kill).
- **UT-082** (state): node-level and loop-level effects for one failure → both fire in own
  scopes; payloads distinguish scope.
- **UT-083** (happy): `no-op` terminal effect fires with no-op semantics.
- **UT-084** (state): no event/effect observable pre-commit — the relay reads only committed
  outbox rows (asserted via tx fault injection at the store fake boundary); delivery_id is a
  pure function of (loop_run_id, source_event_id, trigger, entry_index).
- **UT-085** (happy): `emit` produces `custom_event` with authored kind, bounded payload, event
  identity.
- **UT-180** (happy): pre-binding available before input resolution — paused node's approval
  link renders in effect template.
- **UT-181** (error): approval-requiring tool → `approval_required` effect result; never blocks.
- **UT-182** (state): effect context passes sanitization (planted secret absent from rendered
  `with`).
- **UT-196** (error): render/resolution failure inside the owning transaction → error-as-data
  outbox row (same delivery_id, `{render_error, diagnostic}`), trigger state still commits,
  relay acks it `failed` without tool execution, sibling entries deliver.
- **UT-077** (happy): `on_pause` effect carries resume/approval link bound to exact pause point
  (node+branch+iteration).
- **UT-078** (state): resolved wait link answer = already-decided with decider + timestamp.
- **UT-079** (state): failed notification effect leaves pause fully visible/decidable in
  inventory.

### Liveness, death, silence (`loop_action_liveness.go` evolved; ADR-016)

- **UT-086** (happy): fresh `LastActivityAt` / in-flight tool / transport presence each count as
  life; no duration input exists in the evaluator.
- **UT-087** (state): supervision override at bind: loop-bound session config carries
  warning=0/timeout=0.
- **UT-088** (state): parked time excluded from wall-clock budget accounting; token spend always
  counted.
- **UT-089** (state): transport reconnect blip → degraded-signal state, not death, not flag.
- **UT-090** (state): active stream + stale checkpoint → healthy; staleness visible in progress
  metadata only.
- **UT-091** (happy): confirmed process exit on live node → continuation resume with provenance
  `resumed`, checkpoint carried.
- **UT-092** (boundary): 3 consecutive resumes with zero post-resume evidence →
  `resume_exhausted` attention flag; 4th resume never fires.
- **UT-093** (state): post-resume evidence resets streak to 0 (a week of daemon restarts never
  exhausts).
- **UT-094** (state): death while paused → recorded, no resume; session restored on continue.
- **UT-095** (state): death while awaiting approval → wait untouched.
- **UT-096** (boundary): death before first checkpoint → cold resume, still streak-bounded.
- **UT-097** (ordering): death + cancel race → cancel wins, node closes canceled, no resume.
- **UT-098** (state): ambiguous evidence → silence path only; resume requires deterministic
  death.
- **UT-099** (happy): full silence past window → `silence` flag + event; nothing else changes.
- **UT-100** (happy): any life evidence → flag self-clears + episode recorded.
- **UT-101** (happy): flagged node inspection lists last evidence, window, available verbs.
- **UT-102** (state): in-flight `make gate-full`-class tool with no output → never flagged (hard
  requirement).
- **UT-103** (boundary): window 0 → silence evaluation disabled; death-resume unaffected.
- **UT-104** (ordering): silence flag then confirmed death → death path takes over; episode
  closes into timeline.
- **UT-187** (withdrawn): source-shape assertion removed per round-1 review B-007 — the
  no-duration-kill invariant is owned behaviorally by UT-062, IT-021, and E2E-006 (injected
  clock far-forward with live evidence never fails a node).

### Cancel ≠ kill (`service_node_control.go`, `service_run_cancel.go`)

- **UT-105** (happy): cancel on live node → control row walks `requested → delivering →
  draining → canceled` with provenance at each step; the verb bumps affected cell epochs;
  delivery via prompt cancel; close reason distinct from failure.
- **UT-106** (idempotency): double cancel / cancel after terminal → success no-op / deterministic
  current-state answer.
- **UT-107** (ordering): completion beats cancel → response reports completed.
- **UT-108** (state): grace exceeded → control row stays `draining`, visibly; no automatic
  kill; kill remains an explicit verb.
- **UT-153** (error): every verb's invalid-state answer names actual state + allowed transitions
  (table over verbs × states).
- **UT-154** (concurrency): two concurrent verbs → single CAS winner; loser answer carries
  winner provenance.
- **UT-188** (error): deleted `stop` surface answers as absent at runtime — POST
  `/loop-runs/:id/stop` returns 404 with an unknown-route body, the CLI reports an unknown
  command for `loop stop`, and the registered native descriptor list resolves
  `compozy__loop_stop` as not found (behavioral delete-target guard; E2E-007 covers the
  journey).

### Sub-loop boundary (ADR-016.6)

- **UT-189** (happy): `on_parent_close` default terminate; `cancel` drains child; `abandon`
  leaves child running independently.
- **UT-190** (state): child failure crosses as classified failure {class, child run ref};
  unknown/oversized detail redacts to class (fail-closed).
- **UT-191** (state): child cancel/failure never cancels the parent implicitly.

### Pause + auto-pause (`node_controls.go`)

- **UT-109** (happy): pause excludes node from scheduling/due-scan; rest of generation plans
  normally; provenance persisted.
- **UT-110** (state): paused node excluded from stall/no-progress signature; visible in progress.
- **UT-111** (happy): resume variants apply plain/reset/immediate semantics, each recorded.
- **UT-112** (happy): verb parity — service layer answers identical shapes for CLI/HTTP/UDS
  callers (single BaseHandlers path).
- **UT-113** (idempotency): pause paused / resume unpaused → idempotent success / invalid-state.
- **UT-114** (error): pause on terminal run → invalid-state naming terminal outcome.
- **UT-115** (state): all lanes paused → run paused-dominant; stall arithmetic empty; work clock
  suspended.
- **UT-116** (state): late async result against paused node → recorded, no wake; handled at
  resume.
- **UT-117** (happy): rule match on (class, attempts, target) → pause with rule id + matched
  condition provenance.
- **UT-118** (state): resumed episode does not re-fire same rule without recurrence.
- **UT-119** (error): invalid rule (bad CEL/unknown field) rejected at config write with
  diagnostics.
- **UT-120** (state): first-attempt broad match honored; provenance names rule.
- **UT-121** (ordering): two matching rules → deterministic order, first wins, named.

### Waits (`node_waits.go`; ADR-017)

- **UT-122** (happy): wait node parks cell `waiting`, writes row with identity/kind/age;
  timer sets `resume_at`.
- **UT-123** (happy): `ResumeWait` atomicity — one transaction validates payload, claims,
  transitions wait/output/control, appends provenance+event, and reserves the coordinator task
  run; a fault injected after claim-write aborts the whole transaction (claimed-without-enqueue
  unrepresentable); concurrent claim gets queued/already-resolved with winner provenance.
- **UT-124** (error): payload violating `expect` → rejected with mismatch diagnostic; wait stays
  parked; 3 failures → `intervention_required`.
- **UT-125** (state): two fan-out lanes' waits addressable independently (node+item+generation
  key).
- **UT-126** (state): due timer under run pause → wake parked; fires on resume.
- **UT-127** (happy): ahead arrival consumed on entry by default; `reject` mode rejects with
  diagnostic.
- **UT-128** (happy): waiting inventory row carries loop, run, node, reason, age; sortable
  input ordering deterministic.
- **UT-129** (happy): ladder steps fire in declared order at deadlines; terminal expiry without
  route → `expired_wait` attention.
- **UT-130** (state): ladder effect failure fail-open; wait + age remain listed.
- **UT-131** (ordering): decision mid-ladder cancels remaining steps; record shows both.
- **UT-132** (boundary): zero waits → truthful empty inventory (no phantom resolved rows).

### Target breaker + quarantine (`target_health.go`; ADR-014)

- **UT-133** (happy): 5 consecutive transport failures on one (family,target) → open; bound
  attempts fail fast `target_unavailable` into normal chain.
- **UT-134** (state): healthy targets unaffected while one open (key isolation).
- **UT-135** (happy): half-open probe per 60s interval; success closes; transitions recorded as
  events.
- **UT-136** (state): semantic + handled failures record breaker success, never count toward
  open.
- **UT-137** (happy): same-node failure across 2 consecutive generations → quarantine (not run
  stall); run continues on independent branches.
- **UT-138** (happy): quarantine entry contains classified chain, attempt timestamps, hints,
  target, input ref — diagnosable without logs.
- **UT-139** (happy): requeue clears control row with provenance, plans `requeue`-origin
  generation via succession; bounds apply.
- **UT-140** (state): uncapped watch-loop any-failure arm keeps run-terminal
  `stalled/circuit_breaker` (unchanged).
- **UT-141** (error): requeue on terminal run → invalid-state; entry stays inspectable.
- **UT-142** (state): requeue with open breaker → admitted, fails fast with open-target cause
  immediately visible.
- **UT-143** (state): repeated quarantine episodes append history; caps never bypassed.
- **UT-144** (happy): `on_quarantine` effect carries entry context.
- **UT-145** (error): entry with planted secret in inputs/causes → sanitized before row/surface.

### Admission dedupe (`admission_claims.go`; ADR-017.5)

- **UT-146** (happy): suppression key = (workspace, loop, source_key, event_key); derivation
  stable across restarts.
- **UT-147** (happy): duplicate delivery → structured suppression answer referencing original
  run id; `duplicate_suppressed` diagnostic appended.
- **UT-148** (state): tombstone persists ≥ horizon; sweep removes only past-horizon rows.
- **UT-149** (state): distinct event keys both admit (identity duplicates only — overlap
  explicitly out of scope).
- **UT-150** (state): duplicate after original run terminal → still suppressed within horizon.
- **UT-151** (error): watch-source definition whose source cannot yield `event_key` →
  `CodeWatchIdentityRequired` at validation; no random fallback path exists.
- **UT-152** (state): suppression counters increment per source; queryable.
- **UT-192** (error): runtime `event_key` validation — empty, >256-byte, or non-UTF-8 key in a
  `PollResponse` fails closed before admission with a `watch_identity_invalid` diagnostic; a
  normalized (NFC, trimmed) key admits.

### Config (`internal/config/loops.go` additions)

- **UT-156** (happy): defaults land per grill table — per kind (retry 3/1s/30s, silence 30m,
  streak 3, cost 10000, waits 3×60s, horizon 168h) for both delivery and watch, plus global
  `[loops.breaker]` 5/60s.
- **UT-157** (error): validation — attempts 0..10, `backoff_base <= backoff_max`, positive
  durations, monotonic pairs; `ValidationError` paths machine-parseable.
- **UT-158** (happy): overlay pointer semantics — unset ≠ zero; autopause appends like
  runtime_rules.
- **UT-159** (happy): `TestToolConfigPathPolicy` covers all 20 new agent-mutable paths (18
  per-kind + 2 global breaker); autopause excluded; per-kind breaker paths do not exist.
- **UT-160** (happy): resolution node > loop_config > default for per-kind keys; breaker keys
  are absent from that chain and report the `breaker (global)` source (US-027 AC-3).
- **UT-161** (state): admission-time pin — config change after start does not alter a running
  run's resolved per-kind values; breaker policy is never pinned (global, reload-scoped).
- **UT-162** (error): invalid autopause rule rejected; existing valid rules untouched.
- **UT-163** (state): removed defaults resolve to shipped defaults; chain never resolves empty.

### DSL/lint (`linter_lifecycle.go`, `linter_wait.go`)

- **UT-170** (error): each new code fires on its minimal bad definition —
  `CodeErrorRouteBackward`, `CodeErrorRouteConflict`, `CodeErrorRouteDead`,
  `CodeRetryOnGoalNode`, `CodeTimeoutExceedsDeadline`, `CodeDurationInvalid`,
  `CodeResultContractInvalid`, `CodeEffectShapeInvalid`, `CodeEffectToolUnknown`,
  `CodeWaitShapeInvalid`, `CodeWaitExpiryWithoutPath`, `CodeWatchIdentityRequired` (table
  driven, one row per code).
- **UT-171** (happy): fully-annotated valid definition lints clean; zero-annotation definition
  lints clean (safe default is legal).
- **UT-172** (error): effect entry with both `emit` and `tool` / neither → `CodeEffectShapeInvalid`.
- **UT-173** (happy): `deadline` on goal node → lint error (goal keeps segment `timeout`
  semantics).
- **UT-174** (happy): `wait` params XOR (for/until/event) enforced; `expires.after` required
  when `expires` present.
- **UT-175** (happy): `on_parent_close` valid only on run-loop; closed enum.
- **UT-176** (state): normalize→lint→normalize round-trip stable for all new grammar (YAML
  round-trip).

### Surfaces (contract/CLI/native)

- **UT-183** (happy): `LoopGenerationOutput` projection includes attempt/next_attempt_at/
  failure_class/disposition; absent for legacy statuses.
- **UT-184** (happy): CLI `loop node …` + `loop nodes` render table and `-o json|jsonl` with
  stable field names (golden JSON).
- **UT-185** (happy): native descriptors — 8 new IDs registered, kill tools Destructive,
  `loop_nodes` Read; input/output schema digests recompute deterministically.
- **UT-186** (error): `compozy__loop_status` output schema accepts extended generation object
  (additionalProperties updated) — schema validation over fixture payload.
- **UT-155** (boundary): inventory pagination — stable order, cursor round-trip, truncation
  indicated.
- **UT-193** (state): run-level cancel/kill contract — the request transaction writes
  `cancel_requested`+`cancel_kind` AND projects `requested` onto every live node control row
  with epoch bumps in the same commit; kill transitions `canceled` (`operator_kill`) in the
  service tx for every non-terminal status; cancel on live-node runs drains and the boundary
  closes `canceled`; cancel on queued/watching/needs-approval/paused runs transitions terminal
  directly with deterministic answers; Goal-bearing runs get prompt-lease revocation + binding
  close inside the transaction; `on_canceled` contract effect selected; repeat verbs
  idempotent.
- **UT-194** (idempotency): `loop_run_events.delivery_key` — INSERT OR IGNORE on
  `(loop_run_id, delivery_key)`; same delivery re-appended produces one row; NULL keys
  unconstrained (legacy kinds unaffected).
- **UT-195** (concurrency): `ResumeDeadNode` — node- OR run-scope cancellation pending →
  deterministic no-op loser answer leaving cell AND ledger unchanged; valid death → cell
  transitions to its continuation state + `resumed` ledger disposition + epoch bump + streak +
  binding rotation + exactly one reservation, all in one tx; replay returns the existing
  reservation; a continuation disagreeing with durable cell truth is unrepresentable.

## Integration Tests

### Coordinator lifecycle against real SQLite

- **IT-001**: toolcall returns transport-success/body-failure → boundary classifies
  `payload_declared`, attempt row appended, no retry, escalation plan with repair context.
- **IT-002**: dynamic bad reference mid-run → node fails `authoring`; `loop validate` on same
  definition reports same diagnostic shape.
- **IT-003**: failing tool with hint → next generation's namespace `previous.*` carries
  cause+hint; absorbed variant records hint without injection.
- **IT-004**: broken `stop_when` ends run with predicate-named reason; broken branch condition
  routes through declared error route.
- **IT-005**: route walk — node fails after 3 transport retries → route target runs; success
  dependents skipped; handled disposition persisted; caps incremented.
- **IT-006**: two parallel lanes fail with independent routes → both route; per-node handled
  marks; no cross-lane interference.
- **IT-007**: allow_fail node fails every generation → run completes; absorbed rows per
  generation; downstream refs absent.
- **IT-008**: unannotated failure → `OriginReattempt` succession with classified context from
  generation 1 and from generation N.
- **IT-009**: run-level pause requested during failure handling → handling completes, pause
  lands at boundary.
- **IT-010**: full retry cycle — fail → `retrying` + due row → due-scan wake → fresh
  deterministic run id (`…a2`) → success; ledger shows attempts 1..2; caps/budget counters
  include both.
- **IT-011**: authored agent retry — checkpoint carried into continuation attempt; token spend
  accumulates.
- **IT-012**: authored timeout fires (injected clock) → `attempt_timeout` classified, retried,
  then deadline closes `budget_exhausted`; late completion appends diagnostic only.
- **IT-013**: epoch fencing — pause between schedule and fire → due-scan drops the stale-epoch
  row with diagnostic; resume reschedules under the new cell epoch.
- **IT-014**: effect relay end-to-end — on_error/on_retry/on_quarantine/contract-terminal +
  `emit` custom kind: trigger event + outbox rows commit in one transaction, relay drains
  post-commit idempotently on `delivery_id`, `effect_results` appended, tool effect observed at
  fake tool caller with pre-bound context; two entries on one trigger yield two distinct
  deterministic delivery ids; no hook dispatch originates from the relay.
- **IT-015**: crash-between-commit-and-drain protocol — (a) kill relay after commit, boot-
  started paged drain rediscovers the pending row with the SAME `delivery_id` (digest
  reproduced); (b) lost post-commit nudge → the cycle-driven page still delivers; (c) crash
  after tool execution but before ack → re-drain re-executes the tool (documented at-least-
  once) while result-event appends INSERT OR IGNORE on `delivery_key` (no duplicate events);
  (d) duplicate concurrent drain calls ack exactly once; equal seq values in two different runs
  never collide; replay equals live; observer failure never touches run.
- **IT-016**: cancel matrix — cancel live node (drain), cancel during backoff, double cancel,
  cancel vs completion race, cancel vs death race → dispositions/answers per contract;
  `on_cancel` fired once.
- **IT-017**: quarantine — same node fails 2 consecutive generations → control row + entry;
  independent branch continues to done; gate rerun needing it → needs-attention; requeue →
  `requeue` origin generation; `COUNT(loop_generations) == generation` holds.
- **IT-018**: pause/resume — pause node (rest proceeds), stall signature excludes it, work-clock
  budget suspended while parked, resume reset-attempts zeroes counter, resume immediate fires
  now.
- **IT-019**: approval link flow — `on_pause` effect link resolves pause point; decision applies
  once with identity; late link answers already-decided; unauthorized actor denied + audited.
- **IT-020**: concurrent resume/approve/verb — two writers race on wait claim and on pause verb
  → single winner, loser answers with provenance (BEGIN IMMEDIATE serialization).
- **IT-021**: liveness — long quiet node with in-flight tool never flagged; silence past window
  flags then self-clears on activity; no duration-based failure exists (deleted-path regression
  with clock far-forward).
- **IT-022**: breaker — 5 transport failures open (family,target) via the loop-target
  deadentity instance; sibling target unaffected; retries fail fast; probe closes after health;
  handled/semantic failures never count; state shared across two runs in one workspace under
  the ONE global `[loops.breaker]` policy; custom loop breaker settings provably do NOT alter
  bridge/MCP/sidecar breaker thresholds or probe cadence (separate service instance, shared
  store); daemon restart preserves open/half-open marks and resets pre-threshold streaks
  (documented weaker behavior — asserted, not assumed).
- **IT-023**: auto-pause — configured rule parks node on 2nd transport attempt with rule
  provenance; resume; no re-fire without recurrence.
- **IT-024**: death-resume — kill acpmock session mid-node → confirmed death → `ResumeDeadNode`
  continuation with checkpoint (cell transitioned + `resumed` disposition appended atomically);
  streak increments; post-resume activity resets; 3 no-progress deaths → `resume_exhausted`
  attention; death while parked → no resume; induced races: node cancel vs death AND run cancel
  vs death (cancel wins, cell+ledger untouched), double death detection (one reservation),
  restart between detection and reservation (re-detected, still exactly one continuation).
- **IT-025**: kill — immediate close, ZERO node-trigger effect deliveries, tool interrupt
  ladder invoked, kill event recorded, AND the contract `on_canceled` terminal effect fires
  exactly once (both halves asserted).
- **IT-026**: inventories — waiting/quarantine/attention listing per run/loop/workspace with
  age ordering, pagination, truthful empty states; flags aggregate per workspace.
- **IT-027**: wait lifecycle — timer wait parks, daemon restart, due-scan resumes exactly once;
  fan-out lanes isolated; admission failures ×3 → intervention_required; decision mid-ladder
  cancels steps; restart during admission never double-resumes.
- **IT-028**: dedupe — same event delivered 3× (incl. across restart) → one run + 2 loud
  suppressions with counters; distinct events admit; post-terminal duplicate suppressed within
  horizon; sweep respects horizon.
- **IT-029**: surface parity — same node verb via HTTP, UDS, CLI (`-o json`), and native tool
  returns identical structured shape; invalid states return the same `ReasonError` codes with
  status + body assertions.
- **IT-030**: workspace isolation — node/wait/claim/inventory queries scoped; cross-workspace
  run id → not-found; no leak through SSE payloads.
- **IT-031**: config lifecycle — set family defaults via `compozy config set` (agent-mutable
  paths), start run, change config, verify admission-time pin for per-kind keys; breaker keys
  resolve globally (never pinned) and report the `breaker (global)` source; effective-config
  inspection reports sources.
- **IT-032**: migration suites — fresh apply, reopen-with-data, ahead, integrity, equivalence
  for the five new tables (controls, attempts, waits, admission claims, effect outbox), the
  three rebuilds (outputs status CHECK, generations origin CHECK, dead_entities kind CHECK),
  and the column additions (`loop_run_events.delivery_key` + partial unique index;
  `loop_runs.cancel_requested` + `cancel_kind`); existing rows preserved.
- **IT-033**: two-writer race — snapshot writer vs task-terminal writer on the same output cell
  under `-race` with epoch CAS → no lost lifecycle state, no resurrected stale schedule.
- **IT-034**: CASCADE — deleting a loop run removes controls/attempts/waits/effect-outbox rows;
  admission claims survive (workspace-keyed) until horizon sweep.
- **IT-035**: sub-loop boundary — child failure surfaces as classified failure with child run
  ref (routable); `on_parent_close: cancel` drains child on parent cancel; `abandon` child
  completes independently; child failure never cancels parent.

## End-to-End Tests

Runtime harness (`make test-e2e-runtime`, acpmock; deterministic terminals):

- **E2E-001** (US-008, US-001): transient blip heals — mock tool fails transport once then
  succeeds → run `done`; attempt history shows retry; a body-declared failure variant never
  retries.
- **E2E-002** (US-005): route fallback — primary tool hard-fails → fallback node runs → `done`;
  handled failure never in breaker; caps include the failure.
- **E2E-003** (US-007, US-013): unannotated failure escalates → repair generation cites failure;
  terminal `on_failed` effect fires once on a forced-fail variant.
- **E2E-004** (US-011): `on_error` notification effect delivers rendered context to mock tool;
  effect failure variant leaves run outcome untouched.
- **E2E-005** (US-012): approval journey — gate parks, `on_pause` link posted, approve via link
  → run resumes, decider identity recorded.
- **E2E-006** (US-015/016): long-running node (clock-injected hours) with no limits → healthy;
  kill session process → confirmed death → resume from checkpoint → `done`; minutes lost, not
  restart.
- **E2E-007** (US-018 + delete target): cooperative cancel drains and closes canceled with
  `on_cancel` fired; kill variant immediate — no node-trigger effects, contract `on_canceled`
  terminal effect delivered once; `stop` surface absent (route 404, CLI/native verb gone).
- **E2E-008** (US-019): live repair — pause wedged node, rest proceeds, fix credential (mock),
  resume reset-attempts → `done` without leaving live state.
- **E2E-009** (US-021): durable wait — timer wait, daemon restart mid-wait, exactly-one resume,
  run `done`.
- **E2E-010** (US-022): waiting inventory lists approval with age; authored ladder re-notifies
  then routes timeout path.
- **E2E-011** (US-023): one sick target — its lane fails fast `target_unavailable` while healthy
  lane completes; probe recovery closes breaker and lane succeeds on requeue.
- **E2E-012** (US-024): quarantine journey — repeat offender quarantines, run continues, entry
  diagnosable via CLI, requeue completes run.
- **E2E-013** (US-025): watch redelivery — same event 3× across restart → exactly one run, two
  structured suppressions, diagnostics visible.
- **E2E-014** (US-026): managing-agent journey via native tools only — list inventories, pause,
  resume, requeue, cancel, confirm outcomes; parity asserted against HTTP responses.

Web (Playwright, `make test-e2e-web`):

- **E2E-015** (US-019, US-024 UI): run page shows retrying attempt info, pause/resume controls
  bound to daemon truth, quarantine entry sheet with requeue; inventory views filter/sort by
  age and render truthful empty states.
- **E2E-016** (US-028): authoring journey — open a custom loop in the editor, declare
  `retry` + `on_error` (single absorption mode + one effect) on a node, clear any blocking
  lint errors, Publish succeeds, start a run, and the run page reflects the authored failure
  contract (retrying/effect path observable from daemon truth). Built-in read-only and
  Start-binding allowlist write path are out of this journey (covered by WT-008 / held Spec 3).

Hero path browser coverage is the content-addressed QA scenario `catalog-runform-walk`
(US-029), walked in the qa-execution task with browser evidence; Visual Contract parity for
catalog/run-form/detail is owned by `eng-ui-screenshot` evidence on the hero-path frontend
task — no additional Playwright visual suite.
