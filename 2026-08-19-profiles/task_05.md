---
status: pending
title: Web Switcher, Settings Page, and SymbolPicker
type: frontend
complexity: high
---

# Task 5: Web Switcher, Settings Page, and SymbolPicker

## Overview

Delivers the phase-2 web surfaces: the menubar profile switcher (S1 — right side, immediately before Settings), the Settings → Profiles page (S4), the `SymbolPicker` primitive in `@compozy/ui` (S5 — the sole new primitive), and the rename/archive/delete dialog set (S6/S7), all backed by the task_04 routes with the server-owned remembered choice cached client-side (ADR-014).

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

**Skills**: `eng-design`, `ui-craft`, `impeccable`, `eng-ui-screenshot` (Visual Contract rows), `better-accessibility` (picker/dialog keyboard paths). Execution runs through the `designer` agent contract (execution mode only); read `web/CLAUDE.md` and `packages/ui/CLAUDE.md` before editing.

<requirements>
- MUST reuse before creating: the `@compozy/ui` inventory is `packages/ui/src/primitives.ts` + `packages/ui/src/exports/*.ts` (the 13-line `index.ts` is only a barrel). The switcher composes `CommandSelect*` + the `Avatar` family; identity glyphs extend the `KindIcon` registry pattern and `OwnerAvatar`/`colorsFor` derivation; `GlobalScopeToggle` is never forked. Shadowing an exported name is a blocked lint error.
- MUST ship `SymbolPicker` in `packages/ui` as the only new primitive (tabs Icons | Emojis, search, grid tinted by chosen color, palette + free hex) with story (`src/components/stories/`) and test (`__tests__/` pattern).
- MUST build the domain composites in `web/src/systems/profiles/` per the `_uiux.md` component plan: `ProfileGlyph`, `ProfileSwitcher`, `ProfileCreateDialog`/`ProfileRenameDialog`/`ProfileArchiveDialog`/`ProfileDeleteDialog`, `-profiles-settings-page.tsx` route partial.
- MUST place the switcher on the right of the menubar, immediately before Settings; the left identity cluster (mark → globe → workspace) is untouched; quiet/neutral while only `default` exists (US-007.EC-2, US-010.EC-1).
- MUST treat selection as server state: reads/writes through the selection routes, client store only as cache for instant render; a switch sets the local active view then persists the remembered choice, rolling back on failure; `profile.selection_changed` invalidates the projection and never force-switches an open client (US-010.EC-4).
- MUST render identity color as data (CSS custom property per row/chip) with AA-computed foreground; color never carries meaning alone (name + symbol always present); the signal palette keeps its meanings (needs-setup warning, destructive danger, archived info).
- MUST keep dialogs faithful to the plan payloads (rename tiers, archive paused list + blocked state, delete enumeration, unarchive reactivation list) — no client-side recomputation of what the plan endpoints return.
- MUST honor the default-read contracts (SD-012) per surface: anything beyond them demotes to disclosure; no helper prose under headings.
- MUST be fully keyboard-operable: arrow-navigable switcher, searchable picker, focus-trapped dialogs.
- MUST NOT render per-profile controls for machine facts (truthful UI); remote surfaces render management absent, not disabled (US-032.AC-2).
</requirements>

## Visual Contract

Reference artboards under `docs/design/opendesign/profiles/` are produced by the design pass **before this task executes**; if a named artboard is absent at execution time, stop (blocked-precondition) — never downgrade to prose. Reference parity binds visual language only; runtime truth, `COPY.md`, and the `@compozy/ui` inventory own content/copy/component identity (authorized deltas, SD-007/L-035).

| ID    | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| ----- | -------------------------- | ----------------------------- | -------- | -------- | ---------------------------------- |
| VC-01 | `profiles-switcher.html` — quiet single state | menubar, only `default` exists | 1440×900 | normative | live menubar chrome (host surface) |
| VC-02 | `profiles-switcher.html` — open menu, plural + active mark + boundary sentence + archived count | switcher menu, fixture with 3 profiles (1 archived) | 1440×900 | normative | fixture names/copy → runtime truth + COPY.md |
| VC-03 | `profiles-switcher.html` — "All profiles" state on | switcher with aggregate state active | 1440×900 | normative | none |
| VC-04 | `profiles-settings.html` — profiles page, active list + archived disclosure | `/settings` profiles partial, populated fixture | 1440×900 | normative | Settings host chrome |
| VC-05 | `profiles-settings.html` — create dialog + name validation error | create dialog, invalid-name state | 1440×900 | normative | error copy → `_dx.md` messages |
| VC-06 | `profiles-picker.html` — icons tab + color row | `SymbolPicker`, icons tab | 1440×900 | normative | icon set → product bundled set |
| VC-07 | `profiles-picker.html` — emojis tab + search empty | `SymbolPicker`, emoji tab, empty search | 1440×900 | normative | none |
| VC-08 | `profiles-picker.html` — invalid custom hex | `SymbolPicker`, hex error state | 1440×900 | normative | none |
| VC-09 | `profiles-lifecycle.html` — rename dialog, repo offers pre-checked + dormant placements | rename dialog, plan fixture | 1440×900 | normative | plan data → rename-plan payload |
| VC-10 | `profiles-lifecycle.html` — rename refusals (name taken inline; `default` permanence) | rename dialog error states | 1440×900 | normative | error copy → `_dx.md` |
| VC-11 | `profiles-lifecycle.html` — archive confirm (paused automations) + blocked-by-running | archive dialog, both states | 1440×900 | normative | none |
| VC-12 | `profiles-lifecycle.html` — delete enumeration + owns-work → archive routing | delete dialog, both states | 1440×900 | normative | enumeration → delete-plan payload |
| VC-13 | `profiles-lifecycle.html` — unarchive confirm listing reactivation items | unarchive dialog | 1440×900 | normative | none |

Evidence per row: `.compozy/tasks/profiles/evidence/visual/task_05/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [ ] 5.1 `SymbolPicker` in `packages/ui` (component + story + test), extending the KindIcon-registry and `colorsFor` patterns.
- [ ] 5.2 `web/src/systems/profiles/` scaffolding: types from generated OpenAPI, query keys/options, hooks for list/selection/plans/mutations, MSW fixtures.
- [ ] 5.3 `ProfileGlyph` + `ProfileSwitcher` composition; menubar right-slot wiring in `desktop-menubar.tsx`/`os-menubar.tsx` (left cluster untouched).
- [ ] 5.4 Selection store: server-backed remembered choice + ephemeral active view (per lens), rollback on persist failure, `selection_changed` invalidation.
- [ ] 5.5 Settings → Profiles route partial: active list (glyph/name/work count), archived disclosure, selection-map inspection, "separation, not security" line.
- [ ] 5.6 Dialog set (create/rename/archive/delete/unarchive) bound to plan payloads with refusal states.
- [ ] 5.7 Component tests + Storybook coverage for the new composites; a11y pass (keyboard, focus traps, labels).
- [ ] 5.8 Playwright journeys (assigned E2Es) + Visual Contract evidence bundles for VC-01..13.

## Implementation Details

Consume only task_04's generated types (`web/src/generated/compozy-openapi.d.ts`); no hand-mirrored DTOs. Query keys copy the workspace-qualified pattern.

### Relevant Files

- `packages/ui/src/primitives.ts` + `packages/ui/src/exports/{foundation,listing,editors,conversation}.ts` — reuse inventory (Avatar :27-35 foundation; `CommandSelect*` listing :53-60; `dialogShellClass`/`ConfirmDialog` editors :3-46; `OwnerAvatar`/`colorsFor` conversation :19-22,78-82; `KindIcon` primitives :54-64).
- `web/src/systems/os/components/os-menubar.tsx:21-226` + `desktop-menubar.tsx:183-250` — menubar cluster + shell wiring (right-slot insertion).
- `web/src/systems/workspace/stores/active-workspace-store.ts:14-135` — the client-persist pattern the profile view-state store adapts (remembered choice itself is server state).
- `web/src/systems/workspace/lib/query-keys.ts:1-19` — query-key namespacing pattern.
- `web/src/routes/_app/settings/` — route-partial convention (`-*-page.tsx`).
- `web/src/generated/compozy-openapi.d.ts` — the only DTO source.

### Dependent Files

- `web/src/systems/session/hooks/session-catalog-streams-store.ts:34-50` — generation bump on profile switch (full wiring completes in task_07).
- `packages/ui/src/index.ts` barrels — `SymbolPicker` export (new name, no shadowing).

### Competitor References

- `.resources/hermes/docs/design/profile-builder.md:6-120` — creation-UX conclusions (dashboard-native, identity-first) behind S4/S5.

### Related ADRs

- [ADR-014](adrs/adr-014.md) — remembered choice vs ephemeral active view.
- [ADR-003](adrs/adr-003.md) — per-workspace restore semantics the switcher renders.
- [ADR-005](adrs/adr-005.md) — the All state lives in the switcher (aggregate UI itself is task_07).

## Deliverables

- `SymbolPicker` primitive (story + test) in `@compozy/ui`.
- `web/src/systems/profiles/` composites + Settings page + dialogs, wired to live routes.
- Server-backed selection caching with rollback + invalidation semantics.
- Passing Visual Contract evidence bundles for VC-01..VC-13 **(REQUIRED)**.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] E2E-013 — S1 journey: quiet single → create via switcher (picker) → identity element → switch refilters → boundary sentence → identical workspace list → per-workspace restore.
- [ ] E2E-014 — S4/S5: settings list/create/edit identity, emoji swap, invalid hex inline error, archived disclosure.
- [ ] E2E-016 — S6 rename dialog tiers + declined-repo dormant hint.
- [ ] E2E-017 — S7 archive confirm/blocked + unarchive reactivation list.

Component/unit coverage for the new composites ships with them (vitest + stories) per `web/CLAUDE.md`; no Go cases assigned.

### Web/Docs Impact

- `web/`: this task is the impact — new `web/src/systems/profiles/`, menubar wiring, settings partial, MSW fixtures, stories.
- `packages/site`: none — the capability docs shipped with task_04 (checked: no doc page renders web-only behavior added here).
- QA impact: new scenarios — add content-addressed untested files for switcher create/switch/restore and settings lifecycle dialogs. Walk owned by task_13.
