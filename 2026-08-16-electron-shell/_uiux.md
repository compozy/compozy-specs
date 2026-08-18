# UI/UX Change Map: Electron Desktop Shell Migration

Every UI surface this feature touches: where it lives today, what changes, which states must be designed, and the reference artboard each surface needs. Artboards land under `docs/design/opendesign/electron-shell/` and become the visual contracts the implementation tasks cite.

Companions: `_spec.md` Part I (behavior authority), `_user_stories.md` (states come from ACs/ECs), `_dx.md` (the non-UI half of the surface).

Scope note: the web UI changes here are the **only** web changes in the program (ADR-006). The shell's own chrome (boot overlay) is not a `web/` surface; its reduced role is contracted in `_dx.md` and the app QA scenarios.

## Design constraints (apply to every artboard)

- Daemon truth only (SD-007): both tracks render exactly what `GET /api/settings/update` (extended) reports — including the live operation view (`operation.phase`, `operation.percent`, holder, last error) that backs every progress state below; unknown renders as unknown; the apply affordance is **absent** — not disabled — for managed installs and for the app track when no app is installed. Polling: existing cadence at rest, 2s while `operation` is non-null.
- Signal palette semantic proposal (final call: design pass): `info` = update available / staged, `success` = up to date / updated, `danger` = failed, `warning` = deferred / blocked, `neutral` = unsupported. Matches the existing `updatePill` mapping (`web/src/routes/_app/settings/-general-update-section.tsx:17-36`).
- Menubar indicator obeys calm defaults: hidden when nothing is available; no counts, no pulsing; OS-shell chrome grammar (DESIGN.md §5) since it lives in the menubar.
- Copy: `COPY.md` register; labels reuse the runtime nouns (`runtime`, `app`); no helper prose under headings; update strings stay consistent with the CLI `message` fields in `_dx.md`.
- Keyboard: the indicator is focusable and activates with Enter/Space; the apply action is a standard button reachable in the Settings tab order.
- Browser/app equivalence: every state below renders identically when the SPA is served in a plain browser — zero desktop-awareness in the SPA.

## Surface map

| #   | Surface                                   | Kind   | Core change                                                        | Stories                        |
| --- | ----------------------------------------- | ------ | ------------------------------------------------------------------ | ------------------------------ |
| S1  | Settings → General → Updates section      | modify | Two-track presentation (runtime + app) + apply action + new states | US-015, US-017, US-018, US-029 |
| S2  | OS menubar update indicator               | new    | Discreet available-update indicator → navigates to S1              | US-029                         |

### S1. Settings → General → Updates section

- **Today**: single-track (runtime only) read-only status. `web/src/routes/_app/settings/-general-update-section.tsx:38-171` renders `SettingsGroup "Updates"` with one `SettingRow` ("CompozyOS version"): version + `updatePill` (available/current/updated/failed/deferred/unsupported), retry on refresh error, release-notes link, optional "Next action" recommendation row (mono), optional "Last error" row. View logic: `web/src/systems/settings/lib/update-presentation.ts:13` (`settingsUpdateView`: checking / error / unavailable / snapshot). Data: `useSettingsUpdate` over `GET /api/settings/update` (preloaded at `web/src/routes/_app/-settings-preload.ts:17,30`).
- **Change**: the section presents **both tracks** from the extended daemon payload — a runtime row and an app row (app row exists only when a desktop app is installed on the daemon host) — and gains an **apply action** wherever self-apply is possible (non-managed runtime; installed app). Applying shows live progress (the staged phases the daemon reports); the section is the navigation target of S2.
- **States to design** (from cited stories' ACs/ECs):
  - Checking (today's spinner state preserved) — US-019.AC-2 cadence-driven refresh.
  - Up to date, both tracks (US-015.AC-3).
  - Update available: runtime only / app only / both (US-015.AC-1, US-029.AC-1).
  - Managed runtime: recommendation shown verbatim, **no apply affordance** (US-017.EC-3; today's `installDescription` behavior preserved).
  - No app installed: app row absent entirely — not an empty row (US-029.EC-3).
  - Applying: per-track progress with the daemon's staged phases (`download` % / `verify` / `install` / `start` / `ready`) (US-015.AC-2, US-017.AC-1).
  - App staged (app closed): staged state with "applies on next launch" copy (US-029.EC-2, `_dx.md` staged example).
  - Blocked (single-flight): the in-progress holder named, retry guidance (US-018.EC-1).
  - Failed + rollback: failure with restored-version truth and last-error row (US-016.AC-1, US-017.AC-2).
  - Refresh error: last-known snapshot + danger pill + retry (today's behavior preserved).
  - Browser-equivalence check state: identical rendering when served outside the app (US-029.AC-3).
- **Artboard**: `electron-shell-settings-updates.html`.

### S2. OS menubar update indicator

- **Today**: does not exist. Menubar composite: `web/src/systems/os/components/menubar/os-menubar.tsx` (OS-shell chrome; hydration-status glyph at `web/src/systems/os/components/os-hydration-status.tsx` is the nearest existing pattern for a small truthful status glyph).
- **Change**: a discreet indicator appears in the menubar **only** while the daemon reports at least one available update (either track). Activation navigates to S1 (Settings → General, Updates section). No count, no dropdown, no auto-dismiss animation.
- **States to design**:
  - Hidden (no update available — the default and the vast majority of time) (US-029.AC-2).
  - Available (one or both tracks) — single visual state; detail lives in S1 (US-029.AC-1).
  - Focus-visible state (keyboard) (US-029.AC-4).
  - Applying/staged/failed: indicator stays hidden — those states are S1's job; the menubar never renders progress or errors (calm defaults; US-029.EC-4).
- **Artboard**: `electron-shell-menubar-update.html`.

## Component plan (design → production mapping)

### Rules

- Compose from the existing section: extend `GeneralUpdateSection` rather than forking a parallel component; the two-track layout reuses `SettingsGroup` / `SettingRow` / `SettingValue` from `@/systems/settings`.
- Artboard CSS is a visual contract, never a stylesheet to import; tokens from `packages/ui/src/tokens.css` only.
- The menubar indicator follows OS-shell chrome grammar (the DESIGN.md §5 carve-out) because it lives inside the menubar composite.

### New `@compozy/ui` primitives

None. `Pill` (+ `Pill.Dot`), `Button`, `Spinner` cover every state chip and action; no new generic primitive is justified against `packages/ui/src/index.ts`.

### New domain components

- `SettingsUpdateTrackRow` (`web/src/systems/settings/components/`): one track's row — label, version transition, pill, apply/progress/recommendation control cluster. Composed from `SettingRow`, `SettingValue`, `Pill`, `Button`, `Spinner`. Used twice by S1 (runtime, app).
- `MenubarUpdateIndicator` (`web/src/systems/os/components/menubar/`): the S2 glyph-button; composed from the menubar's existing item primitives + `Pill.Dot`-scale signal. Used by `os-menubar.tsx`.

### Signal & state mapping

| Design state (artboards)   | Primitive + token                                             |
| -------------------------- | ------------------------------------------------------------- |
| Update available / staged  | `Pill tone="info"` (`#8E8EB5` info tint tokens)               |
| Up to date / updated       | `Pill tone="success"` + `Pill.Dot tone="success"`             |
| Failed / refresh failed    | `Pill tone="danger"`                                          |
| Deferred / blocked         | `Pill tone="warning"`                                         |
| Unsupported / managed-only | `Pill tone="neutral"`                                         |
| Applying progress          | `Spinner` (`text-info`) + phase text in `SettingValue mono`   |
| Menubar available glyph    | menubar item primitive + info-tone dot (no count, no pulse)   |
