---
status: pending
title: Read-Scope Enforcement Sweep and Aggregate Reads
type: backend
complexity: critical
---

# Task 6: Read-Scope Enforcement Sweep and Aggregate Reads

## Overview

Executes the Read-Scope Enforcement Matrix row by row: every A-class list/get/stream takes an explicit `ReadScope`, fail-closed, with exactly two modes (scoped, explicit owner-labeled aggregate) — sessions, tasks, loops + outputs, automations, worktrees, attention, usage, observe, network, bridges, notifications, grants, situation, aggregate-by-id deep links, list cursors, and the catalog stream's profile axis. Ships `tools.Scope.ProfileID`, the two read-only native tools, `--all-profiles` on listing verbs, and this capability's docs/skill content. The spec names this sweep the highest-risk surface: a missed read leaks.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

**Skills**: `eng-code-guidelines`, `golang-master`, `eng-contract-codegen-coship`, `eng-test-conventions`, `testing-boss`, `documentation-writer`.

<requirements>
- MUST wire every row of the `_spec.md` Read-Scope Enforcement Matrix; a surface absent from the table may not ship a listing — the matrix is the completion gate.
- MUST thread `ReadScope{ProfileID, AllProfiles}` as explicit parameters through store methods (no context-injected defaults below the API boundary); `ReadScope.Validate` rejects both invalid states before any query runs (fail-closed, zero rows).
- MUST enforce the two-mode contract on single-item gets too: scoped get of a foreign item → 404; `all_profiles=true` get → owner-labeled resource (the deep-link banner contract; never a client-side exception).
- MUST extend the catalog stream's subscriber predicate with the profile axis (workspace + profile; replay included; `all_profiles` labeled; no-filter subscribe impossible by construction).
- MUST include the profile predicate in every list-cursor fingerprint (a cursor never replays across profiles).
- MUST resolve API params uniformly: absent `profile` = `default` at the handler; unknown → `404 profile_not_found`; archived → `409 profile_archived`; `profile`+`all_profiles` → `400 profile_selection_conflict`.
- MUST add `--all-profiles` to listing verbs with owner-labeled rows (`profile_name` in structured output, resolution frame first) and the `--profile`×`--all-profiles` conflict error.
- MUST add `ProfileID` to `tools.Scope` (refreshing every builtin descriptor digest) and ship `compozy__profile_list` + `compozy__profile_current` (read-only, session-derived, shapes per `_dx.md`).
- MUST derive agent-facing scope from daemon-validated session identity; a differing caller-supplied acting profile fails `profile_session_conflict` — rejected, never ignored (D9); agent aggregate reads stay allowed and labeled.
- MUST keep worktrees visible in every profile, owner-tagged (ruled exception), and loop outputs reachable only through an owned run.
- MUST keep network delivery profile-blind (tag ≠ block) and scheduler/budget/spawn primitives profile-blind (Safety Invariants 12–13).
- MUST add the usage/token profile dimension end-to-end (accumulator writes the rebuilt identity; endpoints scoped + labeled aggregate).
- MUST extract instead of appending: `internal/tools/tool.go` (429) and `internal/settings/collections.go` (406) are near the cap.
- MUST ship the capability's docs (scoped vs aggregate reads, `--all-profiles`, deep-link behavior) and official-skill updates in the same task.
</requirements>

## Subtasks

- [ ] 6.1 `ReadScope` threading: store signatures + `Validate` gate for sessions, tasks, loops (+outputs via owned run), automations, worktrees (owner-tagged always-visible), attention/mutes, grants, event_summaries/dead_entities.
- [ ] 6.2 Observe/usage: accumulator identity write, scoped + labeled-aggregate endpoints, per-profile breakdown rows summing to totals.
- [ ] 6.3 Network + bridges + audit listings scoped/labeled per matrix; cross-profile conversation regression fixture.
- [ ] 6.4 Streams: profile axis on the subscriber predicate + replay; generation contract for clients; `CatalogEvent` gains `ProfileID`/`ProfileName`.
- [ ] 6.5 Aggregate assembly: owner-identity join (name/color/icon, archived owners muted-eligible) for rows and single-item gets.
- [ ] 6.6 List cursors: profile predicate in fingerprints across every paged surface.
- [ ] 6.7 CLI: `--all-profiles` on listing verbs, owner fields in all output formats, conflict error.
- [ ] 6.8 `tools.Scope.ProfileID` + native tools (descriptors, schemas, digests) + situation assembler scoping (leak probe).
- [ ] 6.9 API params + spec files + codegen for every touched listing route; extend transport-parity coverage where routes changed.
- [ ] 6.10 Docs + skill updates; QA scenario flags for scoped/aggregate listing behavior.

## Implementation Details

The matrix (spec §Read-Scope Enforcement Matrix) enumerates seam + consumers + canonical test per row — treat it as the work-list. `analysis/10_analysis_profile-scope-inventory.md` is the authoritative A/A-inh class checklist against missed surfaces.

### Relevant Files

- `internal/session/catalog_page.go:41-58` + `session_catalog_stream.go:33-96` — list scope + stream predicate (built in task_01; profile axis added here).
- `internal/listcursor/cursor.go:1-25` — fingerprint contract.
- `internal/tools/tool.go:284-291` — `Scope` (429 lines — extract new files).
- `internal/agentidentity/identity.go:1-25` — daemon-validated identity derivation.
- `internal/situation/service.go:17-35` — `/agent/context` assembler (single leak-visibility point).
- `internal/store/globaldb/queries/observe_overview.sql:2-20` — the single usage accumulator write path.
- `internal/store/globaldb/global_db_observe_retention_tokens.go:52-58` — retention must not delete asymmetrically across profiles.
- `internal/api/httpapi/routes.go:90-97` — observe routes (aggregate candidates); session/task/loop/automation/worktree route files on both listeners.
- `internal/notifications/scope.go:19-45` — cursor scope threading.
- `internal/api/spec/` — param additions per listing route.
- `analysis/10_analysis_profile-scope-inventory.md` — the sweep checklist.

### Dependent Files

- `internal/cli/` listing command files (session/task/loop/automation/worktree/observe) — `--all-profiles` + owner columns.
- `web/src/generated/compozy-openapi.d.ts` — regen; task_07 consumes labels/params.
- `internal/daemon/` — native-tool registration and descriptor digest refresh.

### Competitor References

- `.resources/hermes/docs/profile-routing.md:9-117` — the enforcement posture phase 3 ports (fail-closed, no default fallback); the route table itself stays deferred (ADR-011).

### Related ADRs

- [ADR-015](adrs/adr-015.md) — explicit-parameter enforcement at the store layer; two modes.
- [ADR-005](adrs/adr-005.md) — labeled aggregate as the only widening.
- [ADR-010](adrs/adr-010.md) — organization not access control (worktrees visible; network shared).
- [ADR-011](adrs/adr-011.md) — entry-point ownership for bridge-delivered work (IT-014).

## Deliverables

- Every matrix row wired and green; fail-closed verified at store, stream, cursor, and situation seams.
- `--all-profiles` CLI + labeled aggregate API + aggregate-by-id gets.
- Native tools + `tools.Scope.ProfileID` with refreshed digests.
- Usage/observe profile dimension live.
- Capability docs + skill updates.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] UT-024, UT-077 — CLI conflict; `--all-profiles` JSONL owner fields + frame.
- [ ] UT-072, UT-073, UT-074, UT-075 — Scope threading; native tool shapes; acting-conflict rejection; network-isolation "not supported".
- [ ] UT-080, UT-081, UT-082 — ReadScope.Validate both invalid states; aggregate owner join incl. archived; stream predicate construction.
- [ ] UT-085 — usage accumulator identity + credential-independent attribution.
- [ ] IT-011, IT-012, IT-013, IT-014, IT-015, IT-016, IT-017, IT-018, IT-019, IT-020 — matrix rows: sessions, tasks/loops/automations, attention, bridge-owner inbound, worktrees, volume isolation, loop outputs, workspace catalog parity, labeled aggregate, observe.
- [ ] IT-022 — situation bundle leak probe.
- [ ] IT-023, IT-024, IT-025, IT-026 — stream live filter both axes; replay filter; labeled aggregate stream; cursor fingerprints.
- [ ] IT-060 — cross-profile conversation delivered; ownership by creating side.
- [ ] IT-066 — agent-identity derivation + session-conflict contract + binding unchanged.
- [ ] IT-069, IT-070 — network/bridge/audit listings; labeled aggregate-by-id on both transports.
- [ ] E2E-001, E2E-002, E2E-005 — golden path verbatim; error-surface sweep; aggregate JSONL byte-shape.
- [ ] E2E-009 — in-session native tool + remote-tier mutation refusal.
- [ ] E2E-011 — stream wire test: zero foreign events during cross-profile burst.

### Web/Docs Impact

- `web/`: `web/src/generated/compozy-openapi.d.ts` regen (aggregate labels, params, `CatalogEvent` fields) — consumed by task_07; no components here.
- `packages/site`: scoped-vs-aggregate reads page in the profiles docs section; `--all-profiles` documented on listing verbs (generated CLI docs regen).
- QA impact: new scenarios — add content-addressed untested files for scoped listings, `--all-profiles` labeled output, deep-link owner lookup, and stream isolation; reset any existing session-catalog scenarios whose payload shape changed. Walk owned by task_13.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: native-tool additions (`compozy__profile_list`/`compozy__profile_current`) with descriptors/digests/toolset registration; no other `compozy__*` IDs change (checked: builtin inventory); bridge SDK surfaces unchanged (cache key lands in task_08).
- Agent manageability: session-derived scope for agents, aggregate reads labeled, deterministic conflict errors, native tools read-only by contract — the `_spec.md` Agent Manageability Plan's phase-3 slice.
- Config lifecycle: none — no keys added/changed (checked: `internal/config` untouched except generated docs references).
