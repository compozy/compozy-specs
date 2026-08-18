---
status: completed
title: Desktop-state engine (`internal/clientstate`)
type: backend
complexity: high
---

# Task 1: Desktop-state engine (`internal/clientstate`)

## Overview

Build the daemon's desktop-state engine: a bbolt-backed KV service with per-workspace commit sequencing, tombstoned revisions, atomic multi-op `Apply`, linearized watch subscriptions, workspace-existence gating, and the `[desktop_state]` config section. This package is the ordering and durability foundation every other task's sync semantics stand on — its invariants (1-9 in the TechSpec) are the program's hardest correctness surface.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST implement `internal/clientstate` exactly to the Core Interfaces in `_techspec.md` (§Core Interfaces): `Service` (Get/List/Apply/Watch/PurgeWorkspace), `Op`/`ApplyOptions`, `Subscription` (AsOfSeq/Snapshot/Events/Err/Close), `Entry`/`Event`, generation-fenced `WorkspaceResolver`, and the full sentinel-error set — every signature final, no drift.
2. MUST store to bbolt at `$AGH_HOME/state/clientstate.db` (0600) with the §Data Models layout: `ws:<id>` buckets, `__meta.seq` commit sequence, domain buckets, envelope encoding (rev/seq/flags/updated_at/payload), tombstones removed only by purge.
3. MUST serialize commit→publish through one per-workspace sequencer so hub delivery order equals commit order (invariant 2), and linearize Watch registration+snapshot inside the sequencer window (invariant 3).
4. MUST make `Apply` all-or-nothing in one bbolt tx with consecutive seqs (invariant 4); rev continuity survives delete→recreate via tombstones (invariant 1); CAS honors tombstone revs.
5. MUST enforce slow-consumer eviction with a bounded per-subscription buffer of 256 (invariant 5) without ever blocking writers or sibling subscribers.
6. MUST resolve the workspace via `WorkspaceResolver` on every operation (`ErrWorkspaceNotFound` on unknown/deleted) and serialize purge behind a deletion gate (invariant 9); nothing is created implicitly.
7. MUST add daemon-global config section `[desktop_state]` (`max_value_kib` 256 [1..4096], `max_keys_per_workspace` 512 [16..8192]) with structs, defaults, global overlay, workspace-overlay rejection, validation, `config.toml` example, and tests in the same change.
8. MUST enforce payload validation as size + identifier + JSON-object-shape only — never interpret `os_shell` payload semantics (invariant 11); no query features of any kind (ADR-008 boundary).
9. MUST emit the §Monitoring slog events (payload contents never log) and integrate the bounded-cardinality metrics with the existing observability spine.
10. MUST follow `eng-code-guidelines` (error wrapping `%w`, `errors.Is/As`, slog, context discipline, compile-time interface assertions) and keep every file under the 500-line cap — plan the package split before writing (store, sequencer, hub, service, resolver, config, errors as separate files).
</requirements>

## Subtasks

- [x] 1.1 Package skeleton + sentinel errors + validation (domain/key regexes, size/quota guards, JSON-object check)
- [x] 1.2 bbolt store: open/close lifecycle, bucket layout, envelope codec, `__meta.seq`, tombstone semantics
- [x] 1.3 Per-workspace sequencer: serialized commit→publish, seq stamping, Apply batching (all-or-nothing, consecutive seqs)
- [x] 1.4 Hub + Subscription: linearized register→snapshot→events, AsOfSeq fence, bounded buffers, slow-consumer eviction
- [x] 1.5 WorkspaceResolver integration + deletion gate + PurgeWorkspace (close subs → purge → idempotent re-entry)
- [x] 1.6 `[desktop_state]` config: structs, defaults, ranges, global overlay, workspace rejection, example, validation errors
- [x] 1.7 Observability: slog events + metrics registration
- [x] 1.8 Full assigned test suite (`-race`), including the concurrency cases (UT-070..073, UT-077)

## Implementation Details

New package `internal/clientstate` — the only inward dependency is the `WorkspaceResolver` interface it owns (implemented later by daemon wiring in task_02). Follow `_techspec.md` §Data Models for the envelope byte layout and §Safety Invariants 1-9 as the normative behavior list. Config lands in the existing config package following the current section-struct pattern; find the canonical `config.toml` example file and extend it.

### Relevant Files

- `internal/clientstate/` (new) — store.go, sequencer.go, hub.go, service.go, resolver.go, config.go, errors.go, validate.go + `*_test.go`
- `internal/config/` — existing config section registration pattern to extend with `[desktop_state]`
- `go.mod` — `go get go.etcd.io/bbolt` (never hand-edit)
- `internal/store/` — read-only reference for daemon storage conventions (do NOT couple; boundary per ADR-008)

### Dependent Files

- `internal/daemon/` (task_02) — will construct the service, wire the resolver, own boot/shutdown ordering
- `internal/api/` (task_02) — will consume `Service` + sentinel errors for HTTP/UDS/WS mapping
- `mage` boundaries config — new internal package must be registered in the same commit (SD-008)

### Related ADRs

- [ADR-004](adrs/adr-004.md) — KV + WS decision and the round-1 hardening amendment (seq/tombstones/pumps/resolver)
- [ADR-008](adrs/adr-008.md) — envelope-typed contract on KV; boundary rule; per-workspace quota

## Deliverables

- `internal/clientstate` package implementing every Core Interface signature with invariants 1-9 enforced
- `[desktop_state]` config section with full lifecycle (structs/defaults/validation/example/tests)
- Observability events + metrics per §Monitoring
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-001–UT-021 — store/service CRUD, rev monotonicity, CAS, size/quota limits, domain/key validation, watch ordering/eviction/origin, purge, durability, isolation, config validation
- [x] UT-070, UT-071 — sequencer commit-order delivery + linearized subscribe fence under concurrency (`-race`)
- [x] UT-072, UT-073 — tombstone rev continuity across delete→recreate; atomic Apply batches
- [x] UT-076, UT-077 — WorkspaceResolver rejection on every op; generation-fenced deletion-gate race + idempotent purge + clean recreate
- [x] UT-087, UT-088, UT-089 — JSON-object validation, empty apply atomicity, and closed-service lifecycle

## Success Criteria

- Every assigned test case implemented and passing under `go test -race ./internal/clientstate/...`
- Invariants 1-9 each traceable to at least one passing assertion
- `make lint` clean; no file over 500 lines; boundaries check passes with the new package registered (coordinated with task_02 if wiring lands there)
- Web/Docs Impact: none from this task alone — engine is unexported surface; public exposure, docs, and codegen land in task_02 (checked: no contract/OpenAPI/CLI/web change here)
- Extensibility/Agent Manageability/Config Lifecycle: config lifecycle shipped here (`[desktop_state]` + docs example + tests); agent surfaces land in task_02; no extension/hook/tool surface touched (checked: hook catalog, native toolsets, Host API — unchanged)
- QA impact: the new operator-facing config keys are user-visible — add an `untested` scenario for global limits, workspace-scope rejection, boundary enforcement, validation, and restart durability; public desktop-state transport behavior remains owned by task_02
