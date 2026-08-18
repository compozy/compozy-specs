---
status: completed
title: QA Plan and Session Charters
type: qa-report
complexity: high
---

# Task 6: QA Plan and Session Charters

## Overview

Plans the real-user QA cycle for the shipped feature on the living `docs/qa/` tree: journeys
updated for the new loop semantics, content-addressed scenario files minted/reset, and session
charters selected for execution in task_07.

<critical>ALWAYS READ _techspec.md, every ADR, and every per-task memory file before planning.</critical>

<requirements>
- MUST activate the `qa-report` skill with `qa-docs-path=docs/qa` (tree exists — 703 scenario files).
- Coverage MUST span every public surface touched by tasks 01-05, expressed as scenario `entry_points` on journey-derived rows (not standalone test cases): CLI (`loop status/runs/list/validate/inspect`), HTTP/UDS loop routes + SSE, native tools (`compozy__loop_status`, `compozy__loop_runs`), web run detail/outcome/catalog surfaces, docs pages, and the extension scorer contract.
- MUST mint new `untested` content-addressed scenarios: `LP-ratchet-climb-restore` (metric loop: improve → best; regress → restore; exhaust → best outputs), `LP-dod-reject-retry` (DoD `next_generation` iterates with visible repair context), `LP-revise-repair-context` (revise re-runs producers; prompt carries `previous.verdicts.*`; an explicit in-body `next_generation` starts a fresh pass with `gate_next_generation`).
- MUST reset to `untested`: `docs/qa/scenarios/TA-loop-failure-breaker.md` (re-attempt semantics changed) and `docs/qa/scenarios/LP-loop-run-deep-link.md` (run page payload/UI changed). Dedup against the registry before minting.
- MUST map regression hot spots from `_techspec.md` Safety Invariants (plan atomicity, sanitization, DTO mapping) and ADRs into the cycle's charter selection: targeted tier + one adjacent canary journey (existing loop CRUD/goal journey).
- Output: updated `docs/qa/journeys/`, minted/reset `docs/qa/scenarios/`, cycle charters in `docs/qa/charters/`.
</requirements>

## Subtasks

- [x] 6.1 Update the loops journey flowchart(s) with ratchet/retry/provenance paths.
- [x] 6.2 Mint the three new scenario files (content-addressed ids, `entry_points` across CLI/HTTP/web).
- [x] 6.3 Reset the two affected scenario files to `untested` with reset rationale.
- [x] 6.4 Write the cycle charters (personas, targeted tier + canary journey) into `docs/qa/charters/`.
- [x] 6.5 Cross-check coverage against the TechSpec Agent Manageability Plan and Web/Docs Impact lists.

## Implementation Details

### Relevant Files

- `docs/qa/scenarios/{TA-loop-failure-breaker,LP-loop-run-deep-link}.md` — resets
- `docs/qa/journeys/`, `docs/qa/charters/`, `docs/qa/bugs/` — living tree
- `.compozy/tasks/loops-paper-adoption/_techspec.md` — invariants → hot spots

### Related ADRs

- [ADR-002](adrs/adr-002.md), [ADR-003](adrs/adr-003.md) — the behaviors the scenarios encode.

## Deliverables

- Journeys, scenarios (3 minted + 2 reset), and charters committed on `docs/qa/`.
- Charter selection names the surfaces each session exercises and the evidence it must capture.

## Tests

No `_tests.md` IDs — planning task. The scenario files themselves are the executable contract for task_07.

## Success Criteria

- Every touched public surface appears as an `entry_point` in at least one in-scope scenario
- Scenario/bug ids content-addressed and deduped against the registry
- Charters reference the fix-loop governor and teardown obligations for task_07

## Cycle Charter Selection

Targeted tier, ordered by risk:

1. `CH-loop-repair-succession` — Ada, Interrupt Tour, plan atomicity + sanitization + route scope.
2. `CH-loop-ratchet-truth` — Bruno, Feature Tour, DTO parity + best/provenance + isolation.
3. `CH-runaway-work-bounded` — Ada, Garbage Tour, reset breaker/re-attempt journey.
4. `CH-compozy-run-plain-language` — Cora, Feature Tour, reset deep-link/run-detail journey.
5. `CH-loop-goal-delete` — Bruno, Feature Tour, adjacent Loop CRUD/Goal canary.

The first two are new immutable charters. The remaining three are reused because their missions
still own the changed/reset/canary journeys; debriefs belong in the Task 07 report.

## Taxonomy Coverage

- Journeys and functional: all three feedback branches plus the two reset flows have true end states.
- Experiential: Bruno's run-detail truth and Cora's plain-language deep-link read.
- Edge/error/empty: invalid/non-finite score, no baseline, multi-gate reject, interruption, bounds.
- Cross-cutting: workspace isolation, restart durability, CLI/HTTP/UDS/native/hook/SSE/Web parity.
- Responsive: Loop authoring remains desktop-only by contract; supported run-detail desktop reads
  are covered. No mobile-authoring scenario was invented.

## Completion Evidence

- Tracker materialization: 706 valid scenario files (703 existing + 3 new).
- Three changed/new Mermaid flowcharts parsed with Mermaid 11.16.
- Five selected charters validated for id, persona, journey, canonical tour, time-box, and scenario ownership.
- Public-surface audit named CLI status/runs/list/validate/inspect, HTTP/UDS, native tools, hooks,
  SSE, Web detail/outcome/catalog, site docs, official skill, and extension scorer entry points.
- Gate: full escalation PASS, fingerprint `c5b404c436b3fc5cb395318386cbaa4073d521ea`,
  log `.cache/gate/logs/full-1785591894.log`.

## Compozy Impact Audit

- Native tools: no runtime/schema change; `LP-ratchet-climb-restore` explicitly walks
  `compozy__loop_status` and `compozy__loop_runs` against CLI/HTTP/UDS truth.
- Extensibility and hooks: no implementation change; the ratchet scenario covers extension scorer
  `score`, `loop.gate.post`, `loop.generation.pre`, SSE, and the official skill reference.
- Workspace data isolation: no datum change; ratchet charter requires same-run parity after restart
  plus a denied neighboring-workspace read/cache/event probe.
- Official Compozy skill: no content change in this task; the ratchet scenario names the bundled
  Loop reference as an entry point and compares it with runtime behavior.
- Config lifecycle: no keys changed; all route/ratchet behavior remains definition-declared and
  bounded by existing Loop settings.
