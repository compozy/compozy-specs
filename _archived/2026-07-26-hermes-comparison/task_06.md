---
status: pending
title: "Cost and usage: estimator, provenance buckets, and account-usage surface"
type: backend
complexity: medium
---

# Task 6: Cost and usage: estimator, provenance buckets, and account-usage surface

## Overview

Cost estimation → four-bucket pricing + status/source provenance → conditional account-usage
(L-015 gate). Closes U1 (sessions show no cost unless the agent volunteers one) and makes cost
truthful: estimates vs actuals vs subscription-included, with no fake dollar figures. Account-usage
is verification-first — the fetcher only ships if the spike proves reachability within the native-
CLI auth boundary.

**Authoritative sources:** this suite has no `_prd.md` / `_user_stories.md` / `_tests.md`.
`_techspec.md` and `adrs/adr-006.md` are authoritative. Concrete test cases are inline below
(exact input/condition/expected).

Merges former tasks 19+20+21.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
### Cost estimator (ex-19 / U1)
1. MUST implement `EstimateCost(provider, model, TokenUsage) → CostResult{Amount, Currency,
   Status, Source}` sourcing rates from the merged catalog. Home: `internal/modelcatalog/
   pricing.go` by default; promote to `internal/usage` only if >1 cohesive file accrues
   (techspec §6.4 — YAGNI).
2. MUST enforce strict non-summing precedence: agent-reported `actual` > `estimated` > `included`
   > `unknown`. Double-counting is the named failure mode.
3. MUST wire the estimate into `TokenStatsUpdate` and the task roll-up so `TotalCost` populates
   when the agent is silent; existing CLI/HTTP/extension payloads carry it unchanged in shape.
4. MUST fail open on money surfaces: estimation errors never break session flow.

### Provenance + four buckets (ex-20)
5. MUST add nullable `CostCacheReadPerMillion`, `CostCacheWritePerMillion`,
   `CostReasoningPerMillion` across catalog rows, provider model config, models.dev mapping,
   extension model source, and API/settings contracts (codegen co-ship).
6. MUST add `CostStatus` (`actual|estimated|included|unknown`) and `CostSource`
   (`agent_reported|catalog_config|models_dev|builtin|none`) to `TokenStats`/`TokenStatsUpdate`
   via an append-only global-DB migration, plumbed through API/CLI.
7. MUST classify `included` off provider auth mode (native subscription CLI), never a hardcoded
   provider list.
8. MUST render status-aware cost in web/CLI: `estimated` badge, `included` label, no `$` for
   `unknown` (SD-007).

### Account-usage (ex-21) — verification-first
9. MUST run the verification spike FIRST: can provider-owned CLI credentials reach usage
   endpoints without violating L-015 (native-CLI providers own login state; AGH must not extract
   or re-bind their secrets)? Document per provider with evidence.
10. IF viable for a provider: read-only `AccountUsageSnapshot` fetcher + `agh provider usage
    <provider>` CLI + HTTP endpoint; fail-open; explicit client timeouts (no `http.DefaultClient`).
11. IF not viable: surface truthful `included`/`unknown` only; no fetcher ships; record the
    determination.
12. MUST scope data operator/global — never leak one workspace's spend into another's session
    lists.
</requirements>

## Subtasks (order: estimator → provenance → L-015 spike THEN conditional fetcher)

- [ ] 6.1 Estimation function + precedence logic (+ property tests).
- [ ] 6.2 Wiring: observer update path + task roll-up + surfacing checks; fail-open guarantees +
      docs. (Two-bucket rates here; four-bucket fields arrive in 6.3 — do not block on them;
      `estimated` status is implicit until provenance fields land.)
- [ ] 6.3 Pricing bucket fields across the five mapping surfaces + estimator upgrade to price all
      buckets.
- [ ] 6.4 Status/source fields + append-only migration + plumbing (`global_db_observe.go`);
      `subscription_included` classification off auth mode.
- [ ] 6.5 Web/CLI status-aware rendering (screenshot) + docs.
- [ ] 6.6 Reachability + L-015 boundary spike per provider (evidence-documented) — **before any
      fetcher code**.
- [ ] 6.7 Conditional: fetcher + CLI/HTTP surface for viable providers (+ codegen co-ship); docs
      stating exactly what each provider supports (COPY.md claim standards). If not viable:
      truthful `included`/`unknown` only; record determination.

## Implementation Details

See `_techspec.md` §3.6 / ADR-006. Rates come from the merged live catalog (models.dev + config),
NEVER a hardcoded pricing dict. Migration appends at tail (L-021). TOML keys
`cost_*_per_million` follow the full config lifecycle (SD-011). External account-usage calls carry
explicit timeouts (security invariant). Non-viable path depends on 6.4's `included` classification
for truthful output.

### Relevant Files

- `internal/modelcatalog/pricing.go` (new) — estimator
- `internal/modelcatalog/{types,modelsdev}.go` — buckets
- `internal/config/provider.go` — two-bucket → four-bucket config rates
- `internal/extension/model_source.go` — mapping
- `internal/observe/observer.go` — `TokenStatsUpdate` wiring (no rate join today)
- `internal/task/live.go` — task roll-up where cost stays nil
- `internal/store/types.go`, `internal/store/globaldb/global_db_observe.go` — status/source +
  migration
- spike notes under this task's completion evidence
- conditional: `internal/providers/` or catalog-adjacent fetcher home + `internal/cli/`

### Dependent Files

- `internal/cli/task.go` — existing cost panel now shows populated values
- `internal/api/contract/` + TS — payload fields; conditional snapshot payload
- `web/` session/task cost display
- `skills/agh/` — usage verb docs (conditional)
- `docs/_memory/lessons/L-015-native-provider-auth-boundary.md` — auth-ownership constraint

### Related ADRs

- [ADR-006: Cost estimation and usage provenance](adrs/adr-006.md) — estimator (§1), four-bucket
  + status/source (§2–§4), account-usage L-015 gate (§5)

### Competitor References

- Hermes `usage_pricing.estimate_usage_cost` — join semantics (`analysis/07` §4)
- Hermes 4-bucket + reasoning pricing and cost-provenance fields
- Hermes `agent/account_usage.py` — endpoint shapes + snapshot fields (conditional)

## Deliverables

- Populated `TotalCost` for silent agents; agent-reported cost always wins
- Fully-priced buckets + provenance surfaced truthfully (no fake `$` for subscription/unknown)
- Written viability determination per provider (unconditional)
- Conditional: working read-only usage surface for viable providers
- Every test case in `## Tests` implemented and passing **(REQUIRED for shipped paths)**

## Tests

No `_tests.md` for this suite — concrete inline cases below. Coverage target ≥80% on touched
packages; all Go tests under `-race`. Account-usage tests apply only if the fetcher ships.

### Estimator

- Unit (`internal/modelcatalog` + observer suites — extend):
  - [ ] Known rates + usage fixture → exact expected amount (golden arithmetic; cache tokens at
        input rate until four-bucket upgrade, then per-bucket)
  - [ ] Agent-reported `CostAmount` present → estimation skipped entirely (precedence; no sum)
  - [ ] Missing rate for model → `unknown`, amount nil, no error propagation (fail-open)
  - [ ] Zero-usage update → zero cost, no rate lookup panic
- Integration:
  - [ ] Mock agent emitting usage without cost → session TokenStats carries estimated TotalCost
        end-to-end through HTTP + CLI panels

### Provenance + four buckets

- Unit (modelcatalog + store observe suites — extend):
  - [ ] Usage with cache+thought tokens + full bucket rates → exact expected amount per bucket
  - [ ] Missing reasoning rate only → reasoning priced at output rate or excluded per the
        documented rule (pick one, test pins it)
  - [ ] Subscription-auth provider → `included`, amount nil, source correct
  - [ ] Migration trio: fresh / upgrade-reopen / recorded prefix
- Integration:
  - [ ] End-to-end session → HTTP payload carries status+source; CLI `-o json` matches
- E2E (`make test-e2e-web`):
  - [ ] Web shows `estimated` badge for estimates and `included` (no `$`) for a subscription
        provider fixture

### Account-usage (conditional / verification-first)

- Unconditional:
  - [ ] Viability determination documented with evidence per provider (closes techspec §6.1)
- Unit (fetcher-home package — created only if the fetcher ships):
  - [ ] Usage endpoint 200 fixture → snapshot fields mapped; 401/403 → fail-open with
        deterministic diagnostic (no crash, no retry storm)
  - [ ] Timeout honored (fake slow server) — no unbounded hang
  - [ ] Non-viable provider → surface reports `included`/`unknown`, never errors
- Integration (conditional lane):
  - [ ] `agh provider usage <provider> -o json` + HTTP endpoint parity on the mock provider
- E2E: N/A — operator CLI/HTTP surface; no web UI in this task

## Success Criteria

- Every assigned test case implemented and passing (for shipped paths)
- Coverage ≥80% on touched packages
- U1 closed: silent-agent sessions display non-nil cost sourced from catalog rates
- Zero double-count possible by construction (precedence property test)
- No fake dollar figure anywhere for subscription/unknown cases (SD-007 audit)
- Viability determination documented with evidence; no L-015 violation (no credential
  extraction/re-binding from native CLIs)
- Web screenshot via `eng-ui-screenshot` cited for status-aware cost badges
