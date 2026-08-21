---
status: pending
title: "Foundations: `internal/contracts`, `[calls]` config, store layering"
type: backend
complexity: high
---

# Task 1: Foundations: `internal/contracts`, `[calls]` config, store layering

## Overview

Build the two pure foundations everything else consumes: the `internal/contracts` package — the single result-contract regime (digest registry, admission-pipeline validation, normalization of both contract forms, extraction, budgets, contract-preserving redaction, `SanitizeText`, the relocated `x-compozy-kind` entity walk) — and the complete `[calls]` config section. Close with the store layering fix: the ref/digest helpers move into `internal/contracts` and `internal/store` drops its `internal/loop` imports. Zero behavior change anywhere; this task is the dependency contract for tasks 02 and 03.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. `internal/contracts` MUST expose exactly the Core Interfaces of `_spec.md` (Registry, Verdict, ByteBudget, ResolveBudget/EnforceBudget, ExtractCandidate, RedactPreservingContract, SanitizeText) and MUST import only stdlib + the JSON-schema library (v6, the loop's current engine) + diagnostics-class helpers — never store, loop, task, session, network, or daemon.
2. The registry store MUST be an interface consumed by the package (real SQLite backing lands in task_03); unit tests use a registry-store fake.
3. The admission pipeline order is fixed: secret classification/sanitization on raw bytes FIRST, then validation (verdict + sanitized errors + single-key unwrap) — Safety Invariant 10.
4. Digests MUST be `sha256:<64hex>` over the canonicalized schema; logically identical schemas (reordered keys) pin the same digest; registry rows are schema-only and immutable — budgets are NEVER stored on registry rows (ADR-013, round-3 B-016).
5. Normalization MUST accept both contract forms from `_dx.md` (full JSON Schema and example-shape shorthand) and pin the same digest for equivalent forms; anything else is `call_expect_invalid` with the schema error. Non-object roots are an authoring error.
6. Budgets: `ResolveBudget(override, cfg)` caps per-consumer overrides at `max_budget`; `EnforceBudget` implements `store` (whole payload + bounded preview) and `reject` overflow; stored payloads are never truncated.
7. Validation MUST use a per-digest compiled-schema cache, concurrency-safe (compile exactly once per digest).
8. The repair-prompt builder renders validator issues verbatim (from sanitized output), caps at 10 with `"(+N more)"`, never re-pastes the schema; contract-compile failures classify as contract-fault, not child-fault.
9. The `x-compozy-kind` entity walk moves here from `internal/loop` with behavior parity.
10. `[calls]` config MUST ship complete in this task: struct + `DefaultCallsConfig()` + `Validate()` (path-prefixed errors), overlay merge + profile hook, `tool_surface_calls.go` classification, with keys/defaults exactly per `_dx.md` (`max_depth=3`, `max_batch=8`, `max_children=5`, `max_active_per_root=32`, `idle_ttl="1h"`, results `256KiB`/`4MiB`/`store`, messages `30`/`"30s"`/`50`/`"64KiB"`).
11. Store layering: the six `internal/store` → `internal/loop` helper imports (`OutputRefForPayload`, `ValidateWaitPayload` call sites) are deleted — helpers live in `internal/contracts`; `magefiles/boundaries.go` gains the `internal/contracts` rules in the same change.
12. NO behavior change in any existing pipeline in this task — adoption is task_02.
</requirements>

## Subtasks

- [ ] 1.1 Create `internal/contracts` with the file split decided up front (registry, validation+cache, normalization, extraction, budgets, redaction, sanitize, entity walk, provenance types — one responsibility per file, 500-line cap)
- [ ] 1.2 Implement digest canonicalization + `Registry` (Pin/Resolve/Validate) over a store interface
- [ ] 1.3 Implement the fixed admission pipeline: sanitize-first, then validate with verdicts, single-key unwrap, bounded verbatim issues, repair-prompt builder
- [ ] 1.4 Implement normalization for both contract forms (shorthand ↔ full schema digest parity)
- [ ] 1.5 Implement `ByteBudget`/`ResolveBudget`/`EnforceBudget` with store/reject overflow + bounded previews
- [ ] 1.6 Implement `ExtractCandidate` (fenced/brace-balanced scan, newest-first), `RedactPreservingContract`, `SanitizeText`, and the relocated `x-compozy-kind` walk
- [ ] 1.7 Add the `[calls]` config section end to end: `calls_config.go`, `merge_calls.go` + profile hook, `tool_surface_calls.go`
- [ ] 1.8 Move the shared ref/digest helpers into `internal/contracts`; delete the store→loop imports; update `magefiles/boundaries.go`
- [ ] 1.9 Implement every assigned test case; close on `make gate`

## Implementation Details

Create `internal/contracts/` (new package, leaf). Config lands in `internal/config/` following the network section pattern. Store layering edits touch `internal/store/**` import sites only — no schema changes in this task (the `contract_schemas` table is task_03's migration). Follow `_spec.md` Part II › System Architecture and Core Interfaces; the JSON-schema engine decision and cache semantics are in Technical Considerations › Key Decisions.

### Relevant Files

- `internal/loop/action_schema.go:18-58,102-149,255-294,296-434` — the validation/extraction/repair logic being generalized into contracts (source of truth for behavior parity)
- `internal/config/network_config.go:17,53,82` + `internal/config/merge_network.go:3` + `internal/config/tool_surface_loops.go:13` — config section, overlay, and tool-surface patterns to copy
- `internal/config/loops_test.go` — the per-section config test convention (`calls_config_test.go` is net-new; no `network_config_test.go` exists to extend)
- `internal/store/globaldb/global_db_loop_amendments.go:17-30,179-220` — the amendment/provenance overlay model informing provenance types
- `magefiles/boundaries.go` — boundary rules registry gaining `internal/contracts`
- `internal/session/spawn_wake.go:86-97` — redact-then-bound precedent for `SanitizeText` semantics

### Dependent Files

- `internal/store/globaldb/**` — import sites currently reaching into `internal/loop` for ref/validation helpers (delete target: the six imports)
- `internal/loop/**` — keeps compiling unchanged in this task; adoption rewires it in task_02
- `internal/config/merge.go` / `merge_overlay.go` — profile hook registration for the new section
- `internal/config/config.go` — top-level config struct gaining the `Calls` section

### Competitor References

- `.resources/hermes/tools/delegation_output_schema.py:1-151` — one bounded retry, errors verbatim, no schema re-paste, candidate extraction (the validation/repair/extraction semantics this package encodes)

### Related ADRs

- [ADR-006: One contract package unifies all five structured-output pipelines](adrs/adr-006.md) — the package's reason to exist
- [ADR-010: `internal/calls` + `internal/contracts` package architecture](adrs/adr-010.md) — import boundaries
- [ADR-013: Digest-keyed contract registry with compiled-schema cache](adrs/adr-013.md) — registry identity + cache semantics

### Web/Docs Impact

- `web/`: none — checked surfaces: no API contract, route, or generated type changes in this task; reason: pure internal package + config section with no consumer yet.
- `packages/site`: none in this task — the `[calls]` section block in `content/docs/configuration/config-toml.mdx` ships with task_07 (docs task) once the feature is real; reason: docs describe shipped behavior.
- QA impact: none — no user-visible behavior change yet (`compozy config get/set calls.*` becomes operable, but the keys govern behavior that only exists after task_03..05; the config-lifecycle scenario is flagged by task_05, which completes the observable surface).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked surfaces: extension manifests, hooks, skills/capabilities, tools/resources, registries, bridge SDKs, MCP sidecars; reason: this task adds no extension-visible surface (hooks family + Host API land in task_05).
- Agent manageability: `compozy config get/set calls.*` classification via `internal/config/tool_surface_calls.go` (UT-086) — the only agent-operable surface this task adds.
- Config lifecycle: new `[calls]` section — struct, `DefaultCallsConfig()`, validation, overlay/profile merge, tool-surface classification, tests (UT-083..087). Docs co-ship in task_07. Removed keys: none.

## Deliverables

- `internal/contracts` package complete per the Core Interfaces, boundary-clean (`mage Boundaries` green with its new rules)
- `[calls]` config section fully wired (struct/defaults/validate/merge/tool-surface)
- Store→loop helper imports deleted; helpers living in `internal/contracts`
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] UT-001, UT-002, UT-003, UT-004, UT-005, UT-006, UT-007 — registry pin/resolve/validate, canonicalization, unwrap
- [ ] UT-008, UT-009, UT-010 — repair-prompt builder, contract-fault classification, deterministic issue truncation
- [ ] UT-011, UT-012, UT-013, UT-014 — budget overflow store/reject, `ResolveBudget` cap, exact-boundary behavior
- [ ] UT-015, UT-016 — extraction candidates, newest-valid-wins ordering
- [ ] UT-022, UT-023, UT-024, UT-025, UT-026 — contract-preserving redaction, denylist scrub, split-secret bounded rendering, authoring guidance
- [ ] UT-027 — compiled-cache concurrency (compile exactly once per digest)
- [ ] UT-028 — relocated `x-compozy-kind` walk behavior parity
- [ ] UT-148 — `SanitizeText` clean/redact/reject triad
- [ ] UT-161 — normalization: shorthand pins the same digest as its expanded schema; neither form → `call_expect_invalid`
- [ ] UT-083, UT-084, UT-085, UT-086, UT-087 — `[calls]` defaults, validation rejects, overlay precedence, tool-surface classification, size-string parsing

## Success Criteria

- Every assigned test case implemented and passing (`-race`, `t.Run("Should …")`, `t.Parallel()`)
- `internal/contracts` has zero forbidden imports; boundaries lane green
- `internal/store` no longer imports `internal/loop`; existing loop/store suites still green (behavior parity)
- `make gate` green at close; no file exceeds the 500-line cap
