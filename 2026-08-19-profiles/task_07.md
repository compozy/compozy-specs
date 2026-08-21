---
status: pending
title: Web Aggregate UI (Destination Chip, Owner Labels, Usage, Globe)
type: frontend
complexity: medium
---

# Task 7: Web Aggregate UI (Destination Chip, Owner Labels, Usage, Globe)

## Overview

Completes the phase-3 web surfaces on top of task_06's aggregate reads and task_05's switcher: the fixed "→ default" destination chip + owner toast on creation primitives under "All" (S2), server-scoped listings with owner tags, deep-link owner banners, and profile-named empty states (S3), the usage profile dimension (S10), and the globe/workspace-picker interplay with the profile axis (S11). Profile switches bump stream generations and query-key namespaces — Playwright asserts no stale rows.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

**Skills**: `eng-design`, `ui-craft`, `impeccable`, `eng-ui-screenshot` (Visual Contract rows). Execution runs through the `designer` agent contract (execution mode only); read `web/CLAUDE.md` first.

<requirements>
- MUST render the destination chip as fixed text ("→ default"), part of the default read of every shared creation primitive while the All state is on — never a tooltip, never a picker (ADR-005); success toast names the owner.
- MUST consume server-scoped reads only (no client-side filtering anywhere); aggregate mode adds `ProfileOwnerTag` (glyph+name; archived owners muted) per row; scoped views stay tag-free (calm default).
- MUST bump stream generation and query-key namespace on profile switch; leaving a profile's scope stops its stream writes (US-010.EC-2).
- MUST back deep links to foreign items with the labeled aggregate-by-id read (`all_profiles=true` single get) rendering an owner banner + one-tap switch; surrounding listings stay scoped (US-009.EC-2).
- MUST name the active profile in empty states ("No sessions in Marketing yet", US-009.EC-3).
- MUST adopt the labeled-aggregate read on machine-wide observe/usage dashboards with per-profile breakdown rows (no new dashboard invented, S10).
- MUST implement the S11 interplay: globe = across-workspaces view composing with the profile axis; workspace picker list identical across profiles; profile × globe × workspace axes compose visibly (US-011.AC-2, US-010.AC-4).
- MUST keep worktree rows owner-tagged in every profile (US-009.EC-1).
- MUST reuse `@compozy/ui` primitives (`ProfileOwnerTag`/`ProfileDestinationChip` are `web/src/systems/profiles/` composites over existing primitives — no new `packages/ui` primitive).
</requirements>

## Visual Contract

Reference artboards under `docs/design/opendesign/profiles/` are produced by the design pass **before this task executes**; absent artboard at execution time → stop (blocked-precondition). Parity binds visual language only (SD-007/L-035 authorized deltas).

| ID    | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| ----- | -------------------------- | ----------------------------- | -------- | -------- | ---------------------------------- |
| VC-01 | `profiles-aggregate.html` — labeled rows incl. muted archived owner | sessions listing, All state, mixed-owner fixture | 1440×900 | normative | row content → runtime truth |
| VC-02 | `profiles-aggregate.html` — destination chip on a creation primitive | new-session surface with All on | 1440×900 | normative | host creation surface chrome |
| VC-03 | `profiles-aggregate.html` — owner toast after create-in-All | toast after creation | 1440×900 | normative | toast copy → COPY.md |
| VC-04 | `profiles-aggregate.html` — deep-link owner banner + one-tap switch | foreign session detail view | 1440×900 | normative | none |
| VC-05 | `profiles-aggregate.html` — empty state naming active profile | empty scoped listing | 1440×900 | normative | none |
| VC-06 | `profiles-aggregate.html` — usage breakdown rows per profile | observe/usage dashboard aggregate | 1440×900 | normative | figures → runtime truth |
| VC-07 | `profiles-switcher.html` — globe-on with profile scoped | globe active + profile scoped listing | 1440×900 | normative | none |
| VC-08 | `profiles-switcher.html` — globe-on with All state | globe + aggregate composition | 1440×900 | normative | none |

Evidence per row: `.compozy/tasks/profiles/evidence/visual/task_07/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [ ] 7.1 `ProfileOwnerTag` + `ProfileDestinationChip` composites; owner-tag wiring across session/task/loop/automation/worktree listing rows in aggregate mode.
- [ ] 7.2 Chip + owner toast on every shared creation primitive under All (palette/deep-link parity included).
- [ ] 7.3 Stream-generation + query-key namespace bump on profile switch; scoped-stream teardown semantics.
- [ ] 7.4 Deep-link owner banner backed by the labeled single-get; one-tap switch action.
- [ ] 7.5 Profile-named empty states across listings.
- [ ] 7.6 Usage/observe dashboards: labeled-aggregate adoption + per-profile breakdown rows.
- [ ] 7.7 S11 globe/picker interplay with the profile axis.
- [ ] 7.8 Playwright journeys + Visual Contract evidence bundles.

## Implementation Details

Everything consumes task_06's params/labels through regenerated types. The chip/toast ride the shared creation primitives — no per-surface forks.

### Relevant Files

- `web/src/systems/session/hooks/use-session-catalog-streams.ts` + `session-catalog-streams-store.ts:34-50` — generation-guarded reconnect to bump on switch.
- `web/src/systems/workspace/lib/query-keys.ts:1-19` + `web/src/systems/profiles/` (task_05) — namespacing + composites.
- `web/src/systems/os/components/global-scope-toggle.tsx:22-83` + `os-menubar.tsx:163-188` — globe semantics + workspace chip interplay.
- `web/src/systems/session/components/session-inspector-sections.tsx` — existing cost surface (S10 anchor).
- `web/src/generated/compozy-openapi.d.ts` — aggregate labels/params source.

### Dependent Files

- Listing components across `web/src/systems/{session,tasks,loops,automation,worktree,observe}/` — owner tags, empty states, aggregate consumption.

### Related ADRs

- [ADR-005](adrs/adr-005.md) — chip-not-picker; two read modes; axes compose.
- [ADR-014](adrs/adr-014.md) — active view vs remembered choice on switch.

## Deliverables

- Chip/toast, owner tags, banners, empty states, usage breakdown, globe interplay live on server-scoped reads.
- No stale rows after switch (generation + namespace bump proven in Playwright).
- Passing Visual Contract evidence bundles for VC-01..VC-08 **(REQUIRED)**.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] E2E-015 — S2/S3/S11: labeled rows (archived muted), chip visible, owner toast, palette parity, axes compose, leaving All restores last real profile.
- [ ] E2E-018 — deep-link owner banner + one-tap switch; surrounding lists stay scoped.
- [ ] E2E-019 — empty state names the active profile.
- [ ] E2E-021 — usage scoped figures + labeled aggregate breakdown.

### Web/Docs Impact

- `web/`: this task is the impact — aggregate-mode consumption across listing systems + profiles composites.
- `packages/site`: none — aggregate-read docs shipped with task_06 (checked: no web-only behavior needs a docs page).
- QA impact: new scenarios — add content-addressed untested files for create-under-All chip/toast and deep-link owner banner; reset listing scenarios whose aggregate presentation changed. Walk owned by task_13.
