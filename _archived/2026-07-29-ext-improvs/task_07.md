---
status: completed
title: "CLI UX & operability: diagnostics, status, doctor, events matrix (Phase F)"
type: backend
complexity: medium
---

# Task 7: CLI UX & operability — diagnostics, status, doctor, events matrix (Phase F)

## Overview

Errors help and output confirms (R3/R8): human-mode CLI stops discarding authored diagnostics (11 `WithSuggestedCommand` sites finally reach terminals), success output names the next step, `update_available` becomes passively visible, status renders an honest one-line summary over the four axes with failure counters, the doctor gains an extension probe, jsonl parity closes, and the per-event observability matrix is completed and enforced.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST implement `renderHumanExecutionError` so every authored diagnostic renders `error: <msg>` + detail + `try: <suggested command>` in human mode (the `internal/cli/root.go:216,284-330` degradation dies); structured output unchanged.
- MUST print `✓ <verb> <name>` + a `next:` hint on successful `dev`/`install`/`reload` (golden-tested).
- MUST render the `Update` column in `extension list` (`→ <version>` when `update_available`), surface it in search output, and keep listing non-blocking per the degradation contract (task_05).
- MUST render `consecutive_failures`, `restart_backoff` (payload fields landed in task_05), and a derived one-line summary (`crash-looping (4 failures, backoff 8s)`) in `status` output — axes stay separate in storage (TechSpec Key Decisions).
- MUST add the doctor extension probe reporting crash-loop, missing env, and stale dev origin as distinct diagnostic items.
- MUST close the jsonl gap: `extensionBundle`/`extensionRemoveBundle`/`extensionProvenanceBundle` gain jsonl funcs (the "jsonl formatter is required" failure path disappears — delete target); `-o jsonl` emits one valid object per line on `status`/`remove`/`provenance`.
- MUST fix the path-typo class: a nonexistent local path errors naming the path (stat errors surface — never a slug lookup).
- MUST complete the Monitoring per-event matrix: every lifecycle event from the TechSpec table emits with exactly its required keys, asserted via a call-site event sink; removing any emission fails the matrix; no secret value in any payload (invariants 13–14).
- MUST correct the enable/disable "marketplace operations" mislabel (`internal/cli/extension_state.go:76`).
</requirements>

## Subtasks

- [x] 7.1 Human-mode diagnostic rendering (`root.go` render path) + golden tests for the two most common failures
- [x] 7.2 Success output + next-step hints across dev/install/reload
- [x] 7.3 `Update` column + search surfacing of `update_available`
- [x] 7.4 Status failure counters + derived summary line; mislabel fix
- [x] 7.5 Doctor extension probe (`internal/api/core`, per-probe suite pattern)
- [x] 7.6 jsonl parity funcs (delete the failure path)
- [x] 7.7 Event-emission completion + `events_coverage_test.go` matrix (per-event keys, secret absence)
- [x] 7.8 Implement every assigned test case

## Implementation Details

TechSpec: Agent Manageability (CLI verbs/errors/status discovery), Monitoring and Observability (per-event matrix), Web/Docs Impact (CLI-side items), brief §1 (error renderer evidence) and R8. Phase F gate: CLI golden tests.

### Relevant Files

- `internal/cli/root.go:216,284-330` — the discard path to replace (suite: new `root_test.go`)
- `internal/cli/{format.go,extension_output.go,extension_state.go,extension.go}` — output bundles, columns, jsonl funcs, mislabel (suite: new `extension_output_test.go`)
- `internal/api/core/{extensions.go,marketplace_list.go}` — status payload counters; `update_available` already computed at `marketplace_list.go:401`
- `internal/api/core/extension_doctor_test.go` — new probe suite (pattern: `bridge_doctor_test.go`/`provider_doctor_test.go`)
- `internal/extension/{manager_failure_state.go,marketplace_lifecycle.go,marketplace_update.go}` — counters exposure + event emission call sites
- `internal/extension/events_coverage_test.go` — new matrix suite (UT-061 owner)

### Dependent Files

- `web/src/systems/extensions` status/failure surfaces (task_08 consumes the same payload fields)
- `packages/site` CLI reference regen (task_09)

### Related ADRs

- [ADR-002: First-class dev lane](adrs/adr-002.md) — dev/reload success-path UX
- [ADR-005: GitHub/git-first distribution](adrs/adr-005.md) — update/search surfacing

### Competitor References

- `.resources/flue/packages/cli/src/commands/blueprints.ts:169-253` — agent-caller-aware CLI output (the first-class-agent bar)
- `.resources/flue/blueprints/README.md:52-78` — mechanical update UX precedent

## Web/Docs Impact

Payload additions (`consecutive_failures`, `restart_backoff`) regenerate types consumed by task_08's detail view; no `web/` edits here. Reason-code reference page + CLI reference regen owned by task_09.

**QA impact**: reset the error-UX and status-related ET scenarios (policy-blocked install, daemon-down, status reading) to `untested`; add a content-addressed `untested` scenario for passive update discovery (list/search show updates with zero prior-knowledge flags).

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none structural — checked surfaces: manifests/hooks/tools/registries untouched; event set fixed per the Monitoring matrix.
- Agent Manageability: deterministic errors now reach humans AND stay structured; jsonl parity closes advertised gaps; doctor probe + status summary give agents (and operators) discoverable health; `-o json` carries the same remediation the terminal shows.
- Config Lifecycle: none — checked surfaces: no key changes; `compozy config set` suggestions in diagnostics reference task_02's keys.

## Deliverables

- Every authored diagnostic visible in human mode; success confirms + points next
- Passive update discovery in list/search; honest crash-loop status; doctor probe
- Enforced per-event observability matrix
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] UT-050, UT-051 — policy-blocked + daemon-down human rendering with `try:` lines (golden)
- [x] UT-052 — jsonl parity on status/remove/provenance
- [x] UT-053, UT-054, UT-055 — Update column; failure counters + summary; success `✓`/`next:` (golden)
- [x] UT-060 — doctor probe distinct diagnostics
- [x] UT-061 — per-event coverage matrix via event sink (keys + secret absence; emission removal fails)
- [x] E2E-004 — golden run of the two most common failures on a stamped binary (human stderr carries what `-o json` carries)

## Success Criteria

- Every assigned test case implemented and passing
- Brief R8 check: for the cataloged failure situations, human terminal output contains the remediation/suggested command `-o json` carries
- Update discovery requires zero prior-knowledge flags (R3)
- Coverage matrix fails on any missing lifecycle emission
