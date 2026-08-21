# UI/UX Change Map: Skill Sources

Every UI surface this feature touches: where it lives today, what changes, which states must be designed, and the production mapping. Companions: `_spec.md` Part I (behavior authority), `_user_stories.md` (states come from ACs/ECs), `_dx.md` (the non-UI half of the surface).

**Artboards: none.** Every surface composes the existing settings/menu grammar (`DESIGN.md` + `@compozy/ui` inventory) with no new visual language; no `docs/design/opendesign/` reference is required, and visual-contract capture is therefore not spec-scoped.

## Design constraints (apply to every surface)

- Flat depth; row-list frame idiom `rounded-lg border border-line`; eyebrow class for uppercase table headers; mono for paths and identifiers.
- Signal palette is information only: **warning tint** = truncated root; **danger tint** = broken expose link / escaping entry; origin chips are neutral (mono `Pill`), never colored.
- Truthful UI: rows and counts render only daemon-reported state (`sources[]` envelope). Directory absent renders as an explicit absent state, never a zero that looks measured. Runtime unavailable suppresses counts, keeps policy editable (existing page pattern, `-skills-settings-page.tsx:132-137`). Ineligible affordances are **absent, not disabled** (e.g. no expose action on bundled skills).
- No helper text under headings; `SettingsFieldRow` descriptions only for consequence sentences (≤52ch).
- Copy follows `COPY.md`; config-key provenance chips (`SettingsProvChip`) stay in the Advanced fold.

## Surface map

| #  | Surface                                | Kind   | Core change                                                       | Stories                        |
| -- | -------------------------------------- | ------ | ----------------------------------------------------------------- | ------------------------------ |
| S1 | Settings > Skills — Sources section    | new    | Preset toggle table + custom directories editor + scope behavior   | US-001..005, US-013, US-014    |
| S2 | Session composer `/` picker            | modify | Origin label on skill rows from non-Compozy sources                | US-008, US-006                 |
| S3 | Skills catalog & detail (web)          | modify | Origin attribution + expose action/state on eligible skill detail  | US-010, US-011, US-013         |

### S1. Settings > Skills — Sources section

- **Today**: the page has Engine, Marketplace, Manage, Scope selector, Disabled skills, and an Advanced fold (`web/src/routes/_app/settings/-skills-settings-page.tsx:89-194,214-394`); no source/root control exists. Scope selector covers global/agent only (`:249`); the skills section rejects workspace scope server-side. Section search keywords lack source terms (`web/src/systems/settings/lib/sections.ts:75-80`).
- **Change**: new Sources section rendered first after Engine, global and workspace scopes. Preset rows (compozy always-on without a switch; agents; claude) with pattern paths (mono), daemon-reported per-root state, and a trailing count; custom directories editor beneath (taglist add/remove); workspace scope shows inherited-vs-overridden per key with an explicit override/inherit switch-back; save via the existing inline "applied immediately" controls. Add source keywords to the section search entry.
- **States to design**:
  - Default (US-001.AC-3): compozy row always-on badge; agents on; claude off; counts per root.
  - Directory absent (US-001.EC-1, US-003.EC-1): explicit absent state on the root line, toggle still operable (US-013.EC-4).
  - Truncated root (US-014.AC-1): warning-tint marker + count reflects scanned subset; clears on refresh (US-014.EC-1). All diagnostic content (labels, scanned vs effective counts, skipped links with reasons, collision entries with qualified fallback, verification summary) binds to the daemon per-root diagnostic schema — the UI renders, never derives.
  - Zero optional sources + no custom (US-002.EC-4, US-013.EC-2): "defaults only" presentation via `Empty` in the optional-rows region.
  - Custom editor: add, remove, duplicate rejection inline (US-003.EC-2), scope-invalid path inline (US-003.EC-3).
  - Saving/pending/saved-live feedback matching daemon apply metadata (US-002.AC-4, US-013.AC-5).
  - Save rejected (validation — e.g. stale-client unknown slug, forbidden field — or transport failure): daemon error rendered verbatim, draft preserved, nothing applied; retry available (US-002.EC-3 class).
  - Unreadable root (US-003.EC-6): explicit "not readable" state on the root line, counts omitted entirely (never zeros); toggle stays operable.
  - Workspace scope: inherited (both keys), overridden (per key), switch-back-to-inherit (US-005.AC-2/AC-4, US-005.EC-3).
  - Agent scope: section read-only with the existing scope-policy notice (US-013.EC-3).
  - Runtime unavailable: counts/existence suppressed, toggles editable (US-013.EC-1).
- **Production anchors to copy**: toggle-row table `settings-disabled-skills-section.tsx`; taglist `settings-taglist-field.tsx`; scope gating `use-settings-skills-page.ts:161-202`; draft-kind split `settings-skills-draft-logic.ts`.

### S2. Session composer `/` picker

- **Today**: `SessionComposerCommandMenu` renders Built-in / Agent / Skills sections from the daemon command catalog (`web/src/components/assistant-ui/session-composer-command-menu.tsx:182-240`; grouping `web/src/systems/session/hooks/use-session-commands.ts:92-129`); rows show name/description + scope chip via `commandTrailing` (`web/src/systems/session/lib/session-command-menu-model.ts:242-249`); no origin marker.
- **Change**: skill rows whose origin is not `compozy` gain a discreet mono origin label in the trailing slot (`agents`, `claude`, custom slug). Compozy-native rows unchanged. No layout change; catalog updates keep flowing through the existing SSE invalidation.
- **States to design**:
  - Foreign-origin row with label; native row without (US-008.AC-2).
  - Homonyms from two sources: qualified token forms listed distinguishably (US-008.EC-1, US-006).
  - Source disabled mid-session: rows drop on next catalog refresh (US-002.AC-2, US-008.AC-4 inverse).
  - Long label truncation per existing menu-model conventions.
- **Production anchors**: trailing slot `session-command-menu-model.ts:242`; grouping `use-session-commands.ts:92`.

### S3. Skills catalog & detail (web)

- **Today**: installed/available skills render in the marketplace skills route (`web/src/routes/_app/marketplace.skills.tsx`) with detail components (`web/src/systems/marketplace/components/marketplace-detail-skill.tsx`, `marketplace-detail-skill-installed.tsx`); skill payloads already carry `source`/`dir`/provenance (`web/src/generated/compozy-openapi.d.ts:62726-62764`); no origin column, no expose affordance.
- **Change**: listing rows show origin attribution for non-Compozy skills (same label vocabulary as S2). Eligible skill detail (physical-dir skills) gains an Exposures block: current targets with per-link status, expose action (target picker limited to enabled presets), unexpose per target. Bundled skills show no expose affordance (absent, not disabled). Status rendering: `missing` and `broken` (ours) render danger-tint **with** repair actions (re-expose / unexpose); `foreign_conflict` renders as information only — no action affordances, because CompozyOS never touches foreign entries (US-011.EC-3/EC-4/EC-5).
- **States to design**:
  - No exposures (default) → action only.
  - Exposed healthy (US-011.AC-1/AC-5); exposed missing/broken with repair actions (US-011.EC-3/EC-4); foreign conflict as information only, zero action affordances (US-011.EC-7).
  - Expose action: target picker is a multi-select over enabled presets only; in-flight pending state on the action while the operation runs.
  - Partial failure (multi-target): per-target `results[]` rendered individually — failed targets danger-tint with the daemon's verbatim per-target code, compensated targets marked rolled-back (US-010.EC-1..EC-3, `expose_failed` envelope).
  - Origin attribution on rows (US-013 vocabulary shared with S2).
- **Production anchors**: route `web/src/routes/_app/marketplace.skills.tsx`; detail composites `web/src/systems/marketplace/components/marketplace-detail-skill.tsx` + `marketplace-detail-skill-installed.tsx`; skill hooks/adapters `web/src/systems/skill/hooks/use-skill-actions.ts` + `adapters/skill-api.ts` (expose mutations land beside enable/disable).

## Component plan (design → production mapping)

### Rules

- Compose from `@compozy/ui` and existing settings composites only; domain code in `web/src/systems/settings/` and `web/src/systems/skill/`.
- All counts/paths/states bind to the daemon `sources[]` / `exposures[]` payloads — no client-derived defaults.
- Copy the interaction pattern of `settings-disabled-skills-section.tsx` for the preset table (Switch semantics inverted to "enabled").

### New `@compozy/ui` primitives

None. Everything composes from `Switch`, `Table*`, `Section`, `Pill`, `Empty`, `Input`, `Button`, `Collapsible`, `DropdownMenu`, and the settings composites (`SettingsGroup`, `SettingsFieldRow`, `SettingsTaglistField`, `SettingsInlineSaveControls`, `SettingsAdvancedFold`, `SettingsProvChip`).

### New domain components

| Component | Lives in | Composed from | Used by |
|---|---|---|---|
| `SettingsSkillSourcesSection` | `web/src/systems/settings/components/settings-skill-sources-section.tsx` | `Section`, `Table*`, `Switch`, `Pill`, `Empty`, `SettingsInlineSaveControls` | S1 |
| `SettingsSkillCustomSources` | `web/src/systems/settings/components/settings-skill-custom-sources.tsx` | `SettingsTaglistField` + inline validation slot | S1 |
| `SkillExposePanel` | `web/src/systems/skill/components/skill-expose-panel.tsx` | `Section`/`PropertyRow`, `DropdownMenu`, `Pill`, `Button`, `StatusDot` | S3 |
| Origin label (no new component) | trailing-slot usage of `Pill` (mono) | — | S1, S2, S3 |

### Signal & state mapping

| State | Primitive + token |
|---|---|
| Truncated root | `Pill` with `bg-warning-tint` + warning text token |
| Broken expose link | danger text token + `StatusDot` danger; repair actions as ghost `Button`s |
| Directory absent | muted mono text ("not present"), no signal color |
| Always-on (compozy row) | neutral `Pill` "always on", no `Switch` rendered |
| Inherited vs overridden (workspace scope) | info-neutral text chip via existing scope-notice pattern, not signal palette |
| Origin label | neutral mono `Pill` (never colored) |
