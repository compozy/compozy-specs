---
status: completed
title: "Web loop editor: lifecycle grammar authoring and chrome states"
type: frontend
complexity: high
---

# Task 9: Web loop editor — lifecycle grammar authoring and chrome states

## Overview

Delivers truthful Spec 1 failure-contract authoring in the loop editor: the reliability
envelope, node triggers and contract terminals as effect lists, `wait` and `on_parent_close`
inspectors, and a lint dock where errors gate Publish while warnings never do. Also ships the
chrome states for built-in read-only (fork-before-publish) and publish-rejected (422). This is
the UI that authors the DSL keys task_01 introduces; Start-binding allowlist authoring stays
held (ADR-018 / Spec 3).

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- R1: MUST author Spec 1 reliability envelope fields (`deadline`, `retry` + backoff/non_retryable,
  `result_contract`, `on_error` with route XOR allow_fail + effects) so values round-trip through
  existing PATCH/publish surfaces and reopen identically (US-028 AC-1).
- R2: MUST render six node `on_*` triggers and seven contract terminals (incl. `canceled`) as
  effect lists enforcing emit/tool one-of; empty lists omit zero-count chrome (US-028 AC-2/EC-2).
- R3: MUST expose `wait` inspector (`for`/`until`/`event`, `expect`, `ahead_arrival`, `expires`)
  and run-loop `on_parent_close` (`terminate`/`cancel`/`abandon`) (US-028 AC-3).
- R4: MUST wire lint dock to daemon/linter truth: errors disable Publish; warnings (incl.
  `wait_expiry_without_path`) stay visible and never gate; zero issues render no counter
  (US-028 AC-4).
- R5: MUST ship chrome states — built-in read-only strip + Fork (Publish disabled, Validate ok);
  publish-rejected danger strip on 422 with issue list and version not advanced (US-028 AC-5/AC-6).
- R6: MUST NOT implement Start-binding allowlist write path / hands-free surface switches — held
  Spec 3 (ADR-018); authorized Visual Contract difference vs `loop-editor.html` Start lane.
- R7: MUST verify every Visual Contract row with durable `eng-ui-screenshot` evidence; frontend
  gates via Turbo from repo root only; reuse `@compozy/ui` primitives (no shadow).
</requirements>

## Visual Contract

| ID    | Reference artifact + state | Implementation target + state | Viewport | Fidelity  | Authorized differences + authority |
| ----- | -------------------------- | ----------------------------- | -------- | --------- | ---------------------------------- |
| VC-E1 | `docs/design/opendesign/loops/loop-editor.html` — dirty custom shell at rest | `/loops/$name/editor` — custom loop with unpublished edits | 1440×900 | normative | Start lane / startstrip / hands-free allowlist chrome may keep production behavior (ADR-018 held Spec 3) |
| VC-E2 | `loop-editor.html` — Node tab reliability + six `on_*` + `on_error` | editor Node inspector on action/control | 1440×900 | normative | Field names follow shipped DSL (`on_error` route XOR allow_fail); no invented keys |
| VC-E3 | `loop-editor.html` — Contract tab seven terminals | editor Contract inspector | 1440×900 | normative | `canceled` included; zero-count terminals omit badges (DESIGN-LESSONS L5) |
| VC-E4 | `loop-editor.html` — wait node `await_deploy_ack` | editor wait inspector | 1440×900 | normative | Event block elided only if TechSpec elides that mode |
| VC-E5 | `loop-editor.html` — run-loop `on_parent_close` | editor run-loop inspector | 1440×900 | normative | None |
| VC-E6 | `loop-editor.html` — lint dock error vs warning (`max_fan_out` demo) | editor validation dock | 1440×900 | normative | Errors gate Publish; warnings never; zero counts omit |
| VC-E7 | `loop-editor-states.html` — `state-readonly-source` | editor built-in source | 1440×900 | normative | The reference is its own focused chrome-state lab; production keeps the full editor shell around that state. Runtime source copy uses `marketplace`, the public `LoopSource` value, instead of the prototype-only `builtin` label. |
| VC-E8 | `loop-editor-states.html` — `state-publish-rejected` | editor after 422 publish | 1440×900 | normative | The reference is its own focused chrome-state lab; production keeps the full editor shell and lists every issue returned by the daemon, rather than the reference's single illustrative issue. |

Evidence for each row: `.compozy/tasks/loop-node-lifecycle/evidence/visual/task_09/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 9.1 Schema/field map for Spec 1 envelope + effect lists in editor field libs
- [x] 9.2 Node inspector: reliability fold + six `on_*` effect lists + `on_error` XOR
- [x] 9.3 Contract inspector: seven terminals as effect lists
- [x] 9.4 Wait inspector + run-loop `on_parent_close`
- [x] 9.5 Lint dock severity gating + zero-count omission
- [x] 9.6 Chrome states: built-in read-only/fork + publish-rejected 422 strip
- [x] 9.7 Mocks/fixtures/stories for authoring + chrome states
- [x] 9.8 Vitest WT-005..008 + Playwright E2E-016 + Visual Contract evidence bundles
- [x] 9.9 Flag content-addressed QA scenario `editor-authoring-walk` as `untested`

## Implementation Details

Follow TechSpec Web/Docs Impact (editor bullet) and ADR-018. Extend the existing editor system —
do not invent a parallel authoring surface. PATCH/publish round-trip uses task_07 contract types.

### Relevant Files

- `web/src/systems/loops/components/editor/loop-editor.tsx` — editor shell
- `web/src/systems/loops/components/editor/loop-editor-inspector.tsx` — Contract/Node tabs
- `web/src/systems/loops/components/editor/loop-editor-field.tsx` — field rendering
- `web/src/systems/loops/lib/loop-node-schema.ts` — kind → field set
- `web/src/systems/loops/lib/loop-node-fields.ts` — action/control field defs (retry today)
- `web/src/systems/loops/lib/loop-editor-lint.ts` — client lint presentation
- `web/src/systems/loops/hooks/use-loop-editor.ts` — draft/publish mutations
- `web/src/systems/loops/mocks/{fixtures,handlers}.ts` — linter + publish 422 fixtures
- `web/src/routes/_app/loops.$name.editor.tsx` — route
- `packages/ui/src/index.ts` — primitive inventory (reuse before create)
- `docs/design/opendesign/loops/loop-editor.html` — Visual Contract final
- `docs/design/opendesign/loops/loop-editor-states.html` — chrome-state lab

### Dependent Files

- `web/src/systems/loops/index.ts` — barrel exports
- `web/e2e/__tests__/` — Playwright home for E2E-016
- `web/src/generated/compozy-openapi.d.ts` — regenerated types (task_07)

### Related ADRs

- [ADR-018: Web Visual Contract expansion](adrs/adr-018.md) — editor in Spec 1; Start-binding held
- [ADR-010: DSL failure-vocabulary surface](adrs/adr-010.md) — fields the editor must author
- SD-007 (truthful UI); DESIGN-LESSONS L1/L5/L12

## Web/Docs Impact

This IS the editor web impact. Docs impact: none here (task_11). Do not document Start-lane
authoring as shipping. QA impact below.

## Extensibility / Agent Manageability / Config Lifecycle

None — frontend authoring of daemon-validated DSL only (checked: no new config keys, no new
native tools, no extension protocol changes; agent authoring remains CLI/API from task_07).

## QA impact

Flag new `untested` content-addressed scenario: editor-authoring-walk (declare retry+`on_error`
→ publish → run); walked in task_13 with browser evidence.

## Skills

`react`, `tanstack`, `xstate-store`, `app-renderer-systems`, `storybook-stories`, `vitest`,
`eng-design`, `ui-craft`, `impeccable`, `eng-ui-screenshot` (verification), `tailwindcss`,
`zod` (form/schema boundaries if used).

## References

- TechSpec Web/Docs Impact — editor bullet + Visual Contract authority
- `docs/design/opendesign/loops/DESIGN-BACKLOG.md` §2.2 (grammar closed), §5 (Start held)
- `docs/design/opendesign/loops/DESIGN-LESSONS.md`
- `web/CLAUDE.md` — web-specific dispatch and gates

## Deliverables

- Spec 1 lifecycle grammar authorable in the editor with chrome states
- Durable Visual Contract evidence for VC-E1..VC-E8
  (`.compozy/tasks/loop-node-lifecycle/evidence/visual/task_09/…`)
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**
- QA scenario `editor-authoring-walk` flagged `untested`

## Tests

- [ ] WT-005 — reliability + `on_*` fields round-trip via PATCH/publish mock
- [ ] WT-006 — `on_error` route XOR allow_fail + effect emit/tool one-of gate Publish
- [ ] WT-007 — lint dock: warning never blocks; error blocks; zero count omits counter
- [ ] WT-008 — built-in read-only/fork chrome + publish-rejected 422 strip
- [ ] E2E-016 — author retry+`on_error` → publish → run journey

## Success Criteria

- Every assigned test case implemented and passing (Turbo from repo root)
- Every Visual Contract row has a durable passing evidence bundle
- No Start-binding write path ships; no control renders that the runtime/linter does not model
- `editor-authoring-walk` scenario flagged for task_13
