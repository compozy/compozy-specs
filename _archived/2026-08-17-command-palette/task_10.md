---
status: completed
title: "P8 — NL Fallback, Event Matrix, Integration Close"
type: backend
complexity: medium
---

# Task 10: P8 — NL Fallback, Event Matrix, Integration Close

## Overview

Turns dead ends into delegation and closes the program: the "Ask agent: '{query}'" fallback row (weak/zero-match assembly against `WeightsV1.fallback_weak_match_threshold`, explicit-⏎-only transmission, session spawn with the query as opening prompt), the S15 fallback toggle shipping WITH its behavior, the settings-section completion (IT-015), the cheatsheet/toggle closure (UT-151), and the cross-slice canonical event matrix (IT-033). Flags every QA scenario the program touched for the task_12 walk. This is the last implementation task — the full gate (`make gate-full`) belongs to the loop close after QA.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting; read every ADR in `adrs/` and any `memory/task_NN.md` present
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — fix root causes; frozen `_dx.md`/Part II contracts are binding; surface gaps as clarifications, never invent
</critical>

<requirements>
1. MUST implement the fallback row per F8/US-026: zero-match non-empty query → fallback row only; weak-match (score below `fallback_weak_match_threshold` from the rank-signals `WeightsV1`) → results + fallback together; boundary pinned (UT-140); empty query → never; visually distinct (delegation, not execution); rendered through the normal result assembly (BR: a result row, not a special dialog).
2. MUST transmit nothing before explicit ⏎ (spy-verified — no background classification, no speculative sends; BR-12); ⏎ opens a new session with the workspace's default agent and the query as opening prompt, closes the palette, opens the session window; unresolvable default agent → agent picker path; spawn failure → reason toast with the query preserved; rapid double-⏎ repeat-guarded while deliberate repeats create distinct sessions.
3. MUST ship the S15 fallback toggle WITH the behavior (`fallback_agent_enabled` on `SectionCmdPalette`; off → no fallback row renders; v1 is the single agent target — no ordered list, per the settled agent-only direction) and complete IT-015 (both toggles Live, config echo, 422 on unknown target values).
4. MUST close UT-151: cheatsheet groups extension bindings by source with multi-chord display rules (task_07 data), and the Palette section toggles fallback + resets personalization with scope confirmation.
5. MUST implement IT-033: every low-frequency `cmd_palette.*` event from the Monitoring matrix registered and emitted at its owning domain boundary with required correlation fields (`invocation_id`/`approval_id` on `command.invoked`; `view_session` on session events; `effect_id` on effect-failure logs; workspace everywhere); keystroke/patch paths emit none.
6. MUST run the integration-polish sweep: reasons byte-identical across surfaces (BR-8 spot-audit), settings-affected check (no orphaned key), delete-target sweep (`rg` for every Impact Analysis delete target — zero survivors), docs cross-links complete.
7. MUST flag the program's QA tracker impact for the task_12 walk (flag, not walk — loop rule): verify every scenario named by tasks 01–09 exists in `docs/qa/scenarios/` with `qa_status: untested` (registry-driven root, action panel, inline args + confirmation, extension contributions, agent invoke path, global summon — content-addressed ids, deduped), plus the resets (`ET-web-command-palette-shortcuts`, `ET-window-tab-palette-search`, `ET-palette-nested-views`, `ET-palette-sessions-view-switch`) and a new **NL fallback** scenario from this task.
8. MUST block and surface if a cited artboard is missing at execution time.
</requirements>

## Visual Contract

| ID    | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| ----- | -------------------------- | ----------------------------- | -------- | -------- | ---------------------------------- |
| VC-01 | `docs/design/opendesign/command-palette/command-palette-root-states.html` — zero-match with fallback row only | gibberish query | 1440×900 | normative | None |
| VC-02 | `command-palette-root-states.html` — weak-match with results + fallback together | query at the threshold boundary | 1440×900 | normative | None |
| VC-03 | `command-palette-settings-palette.html` — S15 default state (fallback on + personalization controls) | Settings → Palette | 1440×900 | normative | None |
| VC-04 | `command-palette-settings-palette.html` — fallback disabled state | toggle off; palette renders no fallback row (paired capture) | 1440×900 | normative | None |
| VC-05 | `command-palette-settings-palette.html` — reset confirmation + post-reset feedback | reset flow with scope confirmation | 1440×900 | normative | None |

Evidence for each row: `.compozy/tasks/command-palette/evidence/visual/task_10/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 10.1 `PaletteFallbackRow` + weak/zero-match assembly against the served threshold (boundary-pinned)
- [x] 10.2 ⏎ → session spawn integration (default agent resolution, picker path, failure toast + query preservation, repeat guard) with the nothing-pre-sends spy contract
- [x] 10.3 S15 completion: fallback toggle wired to `SectionCmdPalette` + behavior; reset flow with scope confirmation
- [x] 10.4 UT-151 closure: cheatsheet source grouping + multi-chord rules with extension bindings
- [x] 10.5 IT-033 event-matrix test + any missing registration/emission across the family
- [x] 10.6 IT-015: settings section integration (both toggles, Live, config echo, 422)
- [x] 10.7 Polish sweep: BR-8 reason parity spot-audit, delete-target `rg` sweep, settings-affected check, docs cross-links
- [x] 10.8 QA tracker flags: mint/reset every scenario listed in requirement 7 (content-addressed, deduped) — walks stay with task_12
- [ ] 10.9 Visual Contract evidence bundles VC-01..05

## Implementation Details

Reference `_spec.md` Part II: story→component map US-026, Monitoring and Observability (the event matrix IT-033 asserts), `_uiux.md` S1 fallback states + S15, Business Rules 8, 12.

### Skills

`eng-code-guidelines` · `golang-master` · `eng-test-conventions` · `testing-boss` · `vitest` · `eng-design` · `ui-craft` · `impeccable` · `eng-ui-screenshot`

### Relevant Files

- `web/src/systems/os/lib/ranking/` (task_03) — the threshold consumer; `assembleSections` gains the fallback row slot
- `web/src/systems/session/` session-creation hooks + `routing-coordinator.ts` land-on-session — the spawn + landing path (BR-20 single-path)
- `internal/settings/cmd_palette_section.go` (task_05) — gains `fallback_agent_enabled` diff/apply
- `internal/config/cmd_palette.go` — `fallback_targets` validation already present (task_05); behavior binds here
- `internal/events/names.go` + `registry_base.go` + `internal/observe/observer.go` — the registry/emission surfaces IT-033 sweeps
- `docs/qa/scenarios/ET-palette-nested-views.md`, `ET-palette-sessions-view-switch.md`, `ET-web-command-palette-shortcuts.md`, `ET-window-tab-palette-search.md` — canonical scenarios to reset (never duplicate)
- `web/src/systems/os/components/attention-toast.tsx` + `web/src/lib/user-feedback.ts` — spawn-failure toast path

### Dependent Files

- `web/src/systems/os/` root assembly + stories — fallback states
- `openapi/compozy.json` + generated TS — settings payload completion (`fallback_agent_enabled`)
- `packages/site` palette/config docs — fallback target documentation completes
- `docs/qa/scenarios/` — minted/reset files per requirement 7
- `skills/compozy/` — fallback operation notes (settings path)

### Competitor References

- (No `.resources/` subset — the fallback design is grill-settled product behavior; no competitor port.)

### Related ADRs

- [ADR-003](adrs/adr-003.md) — `fallback_weak_match_threshold` lives in the served `WeightsV1`
- [ADR-006](adrs/adr-006.md) — fallback renders through the single result assembly; settings through the section machinery

### Web/Docs Impact

- `web/`: `PaletteFallbackRow` + assembly integration, S15 completion, cheatsheet grouping closure, stories for VC states.
- `packages/site`: palette/config docs completion (fallback targets), generated references.
- QA impact: flag then hand off — this task mints/resets every scenario in requirement 7 (new: **NL fallback**; verifies the six program scenarios exist untested; resets the four canonical palette scenarios incl. the standing `blocked-verify` on `ET-palette-sessions-view-switch`); ALL walks run in task_12 per the loop contract.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked surfaces: no manifest/hook/SDK change (fallback is core behavior; extension bindings only render in the cheatsheet closure).
- Agent manageability: fallback configuration via `GET|PATCH /api/settings/cmd-palette` + `compozy config get|set cmd_palette.fallback_targets` (both landed earlier; behavior binds now); no new verb — checked against `_dx.md`.
- Config lifecycle: completes the `[cmd_palette].fallback_targets` story — behavior + settings toggle + docs bind to the key task_05 shipped; no new key, none removed.

## Deliverables

- Fallback row + delegation flow live with the nothing-pre-sends guarantee
- S15 complete (toggle with behavior) + UT-151/IT-015 closed
- IT-033 event matrix green across every family the program added
- Polish sweep evidence (reason parity, delete-target zero-survivors, settings-affected check)
- QA scenarios minted/reset per requirement 7 (walks deferred to task_12)
- Visual Contract evidence bundles VC-01..05 **(REQUIRED)**
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] UT-140, UT-141, UT-142, UT-143, UT-144 — zero/weak-match assembly + threshold boundary, pre-send spy, no-default-agent picker, spawn-failure preservation, repeat guard vs deliberate repeats
- [x] UT-151 — cheatsheet source grouping + multi-chord rules; fallback toggle off → no row; reset with scope confirmation
- [x] IT-015 — settings section: both toggles Live, config echo, 422 unknown target
- [x] IT-033 — canonical event matrix with correlation fields; no keystroke/patch events
- [ ] E2E-013 — gibberish → fallback-only → ⏎ → session with query as first prompt; no-default-agent picker variant

## Deferred Tail Evidence

The accepted loop boundary moves VC-01..05 capture and E2E-013 execution to Task 12 and the
workflow tail. Task 10 closes the implementation, focused checks, and QA flags without claiming
visual or end-to-end PASS early.

## Success Criteria

- Every assigned test case implemented and passing
- Transport spy proves zero bytes carry the query before ⏎ (UT-141 pins BR-12)
- Threshold behavior is boundary-exact against the served `WeightsV1` (UT-140)
- IT-033 fails if any Monitoring-matrix event is unregistered, mis-fielded, or emitted from a keystroke path
- Delete-target sweep finds zero survivors of the Impact Analysis list
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence
- `make gate` green (the program's `make gate-full` runs at loop close, after task_12)
