---
status: pending
title: Web sources settings, picker origin chips, and expose panel
type: frontend
complexity: high
---

# Task 6: Web sources settings, picker origin chips, and expose panel

## Overview

Delivers the three web surfaces from `_uiux.md`: the Sources section in Settings > Skills (S1 — preset toggle table, custom directories editor, workspace inherit/override), origin labels on skill rows in the session `/` picker (S2), and the Exposures block on eligible skill detail (S3 — multi-select expose, per-target results, four-state health rendering). Everything binds to the daemon read models shipped in tasks 02/04/05 — the UI renders, never derives.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — no compat shims, no fallbacks, no placeholders (greenfield hard cuts)
</critical>

<requirements>
1. MUST implement S1 (`SettingsSkillSourcesSection` + `SettingsSkillCustomSources` in `web/src/systems/settings/components/`) with every state row from `_uiux.md`: default rows (compozy always-on badge, no Switch), directory-absent, truncated (warning tint + daemon diagnostic binding), defaults-only `Empty`, custom add/remove with inline duplicate + scope-invalid errors, saving/pending/saved-live from daemon apply metadata, save-rejected (daemon error verbatim, draft preserved, nothing applied), unreadable root (counts omitted entirely — never zeros), workspace inherited/overridden per key + switch-back-to-inherit, agent-scope read-only notice, runtime-unavailable degradation (counts suppressed, toggles editable).
2. MUST split `-skills-settings-page.tsx` BEFORE adding the section — it sits at 489 lines against the 500-line cap; extract existing sections/wiring into named components so the new Sources section lands in its own file, none growing past the cap.
3. MUST implement S2: skill rows whose origin is not compozy gain a discreet mono `Pill` origin label in the existing `commandTrailing` slot; native rows unchanged; qualified homonym forms render distinguishably; no layout change; long labels truncate per existing menu-model conventions. Labels come from the daemon catalog spec (task_05) — zero client-side derivation.
4. MUST implement S3 (`SkillExposePanel` in `web/src/systems/skill/components/`): no-exposures default (action only), healthy list, `missing`/`broken` danger-tint WITH repair actions (re-expose / unexpose), `foreign_conflict` information-only with ZERO action affordances, multi-select target picker over enabled presets only, in-flight pending state, partial-failure rendering of per-target `results[]` (failed targets danger-tint with verbatim daemon code, compensated targets marked rolled-back), affordance ABSENT (not disabled) on bundled skills; origin attribution on catalog rows sharing the S2 label vocabulary.
5. MUST follow reuse-before-create: compose from `@compozy/ui` (`Switch`, `Table*`, `Section`, `Pill`, `Empty`, `Input`, `Button`, `Collapsible`, `DropdownMenu`) and the settings composites (`SettingsGroup`, `SettingsFieldRow`, `SettingsTaglistField`, `SettingsInlineSaveControls`, `SettingsProvChip`) — no new `@compozy/ui` primitives (per `_uiux.md` component plan); signal palette per the `_uiux.md` mapping (origin labels neutral mono, never colored); no helper text under headings.
6. MUST extend the draft/mutation pipeline truthfully: new `sources` draft kind in `settings-skills-draft-logic.ts`, workspace tri-state override payload (`{"override": …}` with null-to-inherit) through the adapters/mutations, generated types only (no hand-mirrored DTOs), section search keywords gain source terms (`lib/sections.ts`).
7. MUST add MSW fixtures (typed via `compozyApiMock`) and Storybook stories for the new states, and keep component tests in the per-layer `__tests__/` convention.
8. MUST ship the Playwright journeys (E2E-007..011) with actionability-clean selectors (no `force: true`), extending `web/e2e` fixtures for expose-link and foreign-link manipulation hooks.
9. No Visual Contract section applies: `_uiux.md` declares no named visual reference (surfaces compose the existing settings/menu grammar) — `eng-ui-screenshot` verification is not spec-scoped for this task.
</requirements>

## Subtasks

- [ ] 6.1 Split `-skills-settings-page.tsx` below the cap (extraction-only refactor, no behavior change)
- [ ] 6.2 `SettingsSkillSourcesSection`: preset rows + per-root states (absent/truncated/unreadable/counts) bound to `sources[]`
- [ ] 6.3 `SettingsSkillCustomSources`: taglist editor + inline validation states
- [ ] 6.4 Workspace scope: inherited/overridden per key, override editing, switch-back-to-inherit; agent read-only; runtime-unavailable degradation
- [ ] 6.5 Draft kinds + adapters + mutations for global full-config and workspace tri-state override; save-rejected handling; section keywords
- [ ] 6.6 S2 picker origin labels in the trailing slot (menu model + row rendering)
- [ ] 6.7 `SkillExposePanel`: state rendering (four states + defaults), multi-select expose, pending, partial-failure `results[]`, repair actions, bundled-absent
- [ ] 6.8 Marketplace detail integration (installed skill composite) + origin attribution on rows
- [ ] 6.9 MSW fixtures + Storybook stories for every new state
- [ ] 6.10 Vitest suites (UT-068..073) + Playwright journeys (E2E-007..011)

## Implementation Details

Follow `_uiux.md` end-to-end — it is the surface contract (states, tokens, component plan, production anchors). Bind exclusively to the daemon payloads: `sources[]`/`inherits` (task_02), expose contract (task_04), catalog origin labels (task_05). Truthful UI: absent ≠ zero, unreadable ≠ empty, runtime-unavailable suppresses counts while policy stays editable.

### Relevant Files

- `web/src/routes/_app/settings/-skills-settings-page.tsx` (489 lines — split first) + `web/src/routes/_app/settings/skills.tsx` — page shell
- `web/src/systems/settings/hooks/use-settings-skills-page.ts:122-213` + `hooks/settings-skills-draft-logic.ts` — page state + draft kinds (tri-state lands here)
- `web/src/systems/settings/components/settings-disabled-skills-section.tsx` — toggle-row table pattern to copy (Switch semantics inverted to "enabled")
- `web/src/systems/settings/components/settings-taglist-field.tsx` + `components/index.ts` — custom editor base + barrel
- `web/src/systems/settings/adapters/settings-sections-api.ts:165-199` + `hooks/use-settings-mutations.ts:193-207` + `lib/sections.ts:75-80` — adapter/mutation/keywords
- `web/src/systems/session/hooks/use-session-commands.ts:6,39,47-77,92-129` + `web/src/components/assistant-ui/session-command-menu-model.ts:21,78,100-134,242-249` + `session-composer-command-menu.tsx:34-240` — picker lanes, metadata mapping, trailing slot
- `web/src/routes/_app/marketplace.skills.tsx` + `web/src/systems/marketplace/components/marketplace-detail-skill.tsx` + `marketplace-detail-skill-installed.tsx` + `hooks/use-marketplace-detail-skill-manage.ts` — S3 anchors
- `web/src/systems/skill/adapters/skill-api.ts` + `hooks/use-skill-actions.ts` + `lib/skill-formatters.ts` + `lib/query-keys.ts`/`query-options.ts` — expose mutations beside enable/disable; origin-label formatting
- `web/src/systems/skill/mocks/{handlers.ts,fixtures.ts}` + `web/src/storybook/openapi-msw.ts` — typed MSW
- `web/src/generated/compozy-openapi.d.ts` — generated types (source of every DTO)
- `web/src/systems/settings/routes/settings-skills.stories.tsx` + `web/src/components/assistant-ui/__tests__/session-thread.test.tsx:2202-2434` — story + composer test anchors
- `web/e2e/__tests__/` + `web/e2e/fixtures/` — Playwright layout + fixture hooks
- `packages/ui/src/index.ts` + `packages/ui/src/tokens.css` + `DESIGN.md` — primitive inventory + token grammar (reuse-before-create gate)

### Dependent Files

- `web/src/systems/settings/hooks/__tests__/use-settings-skills-page.test.tsx` — extends for sources draft kinds
- `web/src/systems/skill/**/__tests__/*` — per-layer test extensions
- `web/CLAUDE.md` dispatch — skills to activate before writing code

### Related ADRs

- [ADR-006](adrs/adr-006.md) — workspace override with web management · [ADR-011](adrs/adr-011.md) — expose from the skill's web surface · [ADR-013](adrs/adr-013.md) — origin chips in picker + settings

### Web/Docs Impact

- `web/`: this task IS the web impact — systems/settings (section + hooks + adapters), systems/skill (panel + mutations + formatters), assistant-ui picker trailing slot, MSW fixtures, stories, Playwright fixtures.
- `packages/site`: none in this task — docs land in task_07.
- QA impact: flag (walk deferred to task_09) two new content-addressed `untested` scenarios: (a) Settings > Skills sources section — toggle live counts, custom add/remove with inline errors, workspace inherit/override/switch-back, save-rejected draft preservation, unreadable/truncated rendering; (b) skill detail expose panel + picker chips — expose multi-select, partial failure per-target rendering, foreign-conflict no-affordance, origin labels in composer.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — web-only composition over existing daemon contracts (checked: no extension surface, no new primitives in `@compozy/ui`).
- Agent manageability: none — this is the operator surface; agent parity shipped in tasks 01/02/04 (checked: no new web-only capability is introduced — every action here has a CLI/API equivalent).
- Config lifecycle: none — consumes existing keys through the settings API (checked: no new keys, no defaults change).

## Deliverables

- S1 Sources section + custom editor with every `_uiux.md` state, page split under the line cap
- S2 origin labels in the picker trailing slot
- S3 `SkillExposePanel` with four-state rendering, multi-select expose, partial-failure results
- Draft/adapter/mutation pipeline for global + workspace tri-state saves; MSW + stories
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there. Owning suites: Vitest per-layer `__tests__/` (settings hooks + new components, skill system, composer menu), Playwright `web/e2e`. Frontend gates run through Turborepo from the repo root only.

- [ ] UT-068, UT-069, UT-070, UT-071 — S1 states (rows/counts/absent/unavailable; truncated/defaults-only/collisions; workspace + agent scopes; custom editor errors + save rejection)
- [ ] UT-072 — picker origin labels (present on foreign, absent on native)
- [ ] UT-073 — expose panel full state matrix (four states, multi-select, pending, partial failure, bundled-absent)
- [ ] E2E-007 — toggle claude → applied live → counts → picker drop
- [ ] E2E-008 — custom source add/duplicate error/remove
- [ ] E2E-009 — workspace scope inherit → override → switch back (cross-workspace check via API)
- [ ] E2E-010 — composer `/` shows origin label on absorbed skill, none on native
- [ ] E2E-011 — expose → healthy; fixture link deletion → missing + repair; foreign link → conflict with no affordances

## Success Criteria

- Every assigned test case implemented and passing; web lanes green via `make gate` (turbo-routed)
- Zero `compozy-ui-reuse/no-shadow-ui-primitive` violations; no new `@compozy/ui` primitives
- No file over the 500-line cap after the page split; new components one-responsibility-per-file
- Every rendered count/state traceable to a daemon payload field (no client-derived defaults), including the three "never lie" states: absent, unreadable, runtime-unavailable
