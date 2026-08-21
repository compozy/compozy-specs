---
status: completed
title: "P5 — View System: Vocabulary, Patch Engine, Four Kinds, Domain Views"
type: backend
complexity: high
---

# Task 6: P5 — View System: Vocabulary, Patch Engine, Four Kinds, Domain Views

## Overview

Delivers the Go-owned view vocabulary (JSON tags = frozen wire, ADR-007) with validation, per-element `requires`/`fallback`, caps, and the revision-fenced RFC 6902 patch engine + generated TS types; generalizes the shipped view shell to all four kinds (List, Detail, Form, Grid) under unchanged stack semantics; and registers a curated built-in view for every list-bearing domain on the Sessions grammar. Tier-1 view routes (`GET /views/{id}` + patch stream) land on both transports — extension fixtures that exercise them arrive with task_07.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting; read every ADR in `adrs/` and any `memory/task_NN.md` present
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — fix root causes; frozen `_dx.md`/Part II contracts are binding; surface gaps as clarifications, never invent
</critical>

<requirements>
1. MUST implement the complete vocabulary structs exactly as Part II Core Interfaces freezes them (`ViewPayload`/`ViewChrome`/`Section`/`Row`/`RowAction`/`Chip`/`EmptyState`/`Pagination`/`DetailBody`/`FormBody`/`FormField`/`GridBody`/`GridTile`/`Effect` union/`PatchOp`/`Badge`/`Image`/`MetaField`/`Confirmation`) — JSON tags are the wire; no internal second schema; `RowAction` validates exactly-one-of `Action|Handler|SubmitForm` (SI-18) and `Destructive ⇒ Confirmation`.
2. MUST validate payloads daemon-side before serving (path-naming errors), enforce the v1 resource caps as named constants (rows/mount cap with exact `showing N of M`, detail truncation markers, ≤500 rows/patch, ≤256 KiB), apply Adaptive-Cards-style versioning: per-element `Requires`/`Fallback` (incl. `"drop"`), unknown elements → Null-Object renderer + capability-gap telemetry, unknown kinds → honest frame (SI-12).
3. MUST implement the patch engine: `ViewPatch{From,To,Ops}` applies only when `From` equals current revision; any gap triggers full-payload resync; out-of-order patches never apply (SI-2); property tests pin patch application.
4. MUST generalize the shipped view shell to render all four kinds under identical stack chrome with the fixed semantics (push/pop, per-level search, breadcrumb ≤3, ⌫-on-empty, Esc-closes-all, reopen-at-root, depth-keyed fresh mounts, selection survival) — BR-7; "view unavailable" frame for dead ids naming the extension.
5. MUST implement kind renderers per S3–S6: Detail accessory pane (selection-driven, focus stays in list, sanitized markdown with plain-text degradation), Form (typed fields in declared order, inline validation, first-invalid focus, submit → command action, discard-on-pop, password masking, empty-dropdown declared hint), Grid (sectioned 2D arrow nav, ⏎/panel parity, placeholder tiles, list empty grammar), with `@tanstack/react-virtual` beyond the mount cap.
6. MUST register a curated built-in view per list-bearing domain (sessions kept + worktrees, tasks, loops, jobs, triggers, agents, bridges, knowledge, vault names-only, network channels, marketplace, extensions) — Sessions grammar generalized (chips with truthful counts, single-select, one-keystroke clear, live refresh with selection survival), shared status-tone dictionary + attention comparator mandatory, no view-local tone maps (US-010.AC-2/AC-3); built-ins construct the normalized model natively from their TanStack queries (Key Decisions — only extension views cross the wire).
7. MUST land Tier-1 view routes on both transports per `_dx.md`: `GET /api/cmd-palette/views/{id}` (validated envelope) + `GET .../views/{id}/stream` (revision-fenced patches; `after>0` requires `stream_epoch` → 400; fence mismatch → full resync) + OpenAPI/TS + parity rows; generated TS vocabulary types co-ship (round-trip fixtures per UT-040 family).
8. MUST block and surface if a cited artboard is missing at execution time.
</requirements>

## Visual Contract

| ID    | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| ----- | -------------------------- | ----------------------------- | -------- | -------- | ---------------------------------- |
| VC-01 | `docs/design/opendesign/command-palette/command-palette-view-shell.html` — List kind under stack chrome at depth ≥2 | Sessions/tasks view pushed with breadcrumb | 1440×900 | normative | None |
| VC-02 | `command-palette-view-shell.html` — Detail, Form, Grid under identical chrome | each kind pushed (seeded views) | 1440×900 | normative | None |
| VC-03 | `command-palette-view-shell.html` — view-unavailable frame naming the source | dead view id frame | 1440×900 | normative | Extension naming exercised fully in task_07 (fixture) — seeded source name here |
| VC-04 | `command-palette-view-shell.html` — loading + timeout/retry frames | slow-source states | 1440×900 | normative | None |
| VC-05 | `command-palette-domain-list.html` — tasks exemplar (chips + truthful counts + state badges) | Tasks domain view | 1440×900 | normative | None |
| VC-06 | `command-palette-domain-list.html` — empty-with-filter naming the active filter | zero-match chip state + one-keystroke clear | 1440×900 | normative | None |
| VC-07 | `command-palette-domain-list.html` — cold-cache loading | domain view opened before its app ever visited | 1440×900 | normative | None |
| VC-08 | `command-palette-domain-list.html` — vault names-only rows | Vault domain view | 1440×900 | normative | None |
| VC-09 | `command-palette-domain-list.html` — detail pane populated + neutral empty | List.Item.Detail beside rows | 1440×900 | normative | None |
| VC-10 | `command-palette-form-grid.html` — form pristine + per-field invalid with first-invalid focus | form view states | 1440×900 | normative | None |
| VC-11 | `command-palette-form-grid.html` — submit-failed with preserved values + password masking + empty-dropdown hint | form error states | 1440×900 | normative | None |
| VC-12 | `command-palette-form-grid.html` — grid populated + placeholder tile + empty + overflow | grid view states | 1440×900 | normative | None |

Evidence for each row: `.compozy/tasks/command-palette/evidence/visual/task_06/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 6.1 Vocabulary structs + validation + caps + requires/fallback + Effect union (`internal/cmdpalette/view*.go`, file split up front) + generated TS + round-trip fixtures
- [x] 6.2 Patch engine: fence check, resync trigger, application property tests
- [x] 6.3 `ViewService.ResolveView`/`OpenSource` (Tier-1 read-only source resolution; validate-time read-only risk class enforcement lands with the manifest in task_07 — the service enforces it at open here)
- [x] 6.4 Tier-1 routes + patch stream (epoch guard) on both transports + OpenAPI/TS + parity rows
- [x] 6.5 Shell generalization: kind-dispatched bodies under one chrome; unavailable/loading/timeout frames; registry-driven view resolution completes
- [x] 6.6 `PaletteDetailPane`, `PaletteFormView`, `PaletteGridView` renderers + sanitization + virtualization
- [x] 6.7 Domain-view template (generalized Sessions grammar) + per-domain chip sets/rows for every list-bearing domain
- [x] 6.8 Form-submit → command-action dispatch (IT-005 seeded provider path)
- [x] 6.9 Generated API docs and stories for every VC state; the extension-facing owning page is created and completed in task_07
- [ ] 6.10 Visual Contract evidence bundles VC-01..12 — deferred to the task_12 QA walk by the accepted loop plan

## Implementation Details

Reference `_spec.md` Part II: Core Interfaces (vocabulary + `ViewService`), API Endpoints (views routes), Key Decisions (built-ins render natively; virtualization), Assumptions (caps, view open budget), Safety Invariants 2, 12.

### Skills

`golang-master` · `eng-code-guidelines` · `eng-contract-codegen-coship` · `eng-test-conventions` · `testing-boss` · `vitest` · `eng-design` · `ui-craft` · `impeccable` · `eng-ui-screenshot` · `better-layout`

### Relevant Files

- `web/src/systems/os/lib/palette-view-stack.ts` + `hooks/use-os-palette-view-stack.ts` + `components/os-palette-view-shell.tsx` + `os-palette-view-stack.tsx` + `os-palette-breadcrumb.tsx` + `os-palette-footer.tsx` — the shipped stack mechanics this task generalizes (breadcrumb ≤3, selection survival)
- `web/src/systems/os/hooks/use-os-palette-sessions-view.tsx` + `lib/palette-session-filters.ts` + `components/os-palette-session-{chips,row}.tsx` + `os-palette-view-note.tsx` — the Sessions grammar template (150-row cap precedent)
- `web/src/lib/status-tone.ts` + `compareAttentionFirst` — mandatory shared dictionaries (UT-133 tone parity)
- `web/src/components/assistant-ui/hooks/use-thread-scroll-controller.ts` — the only in-repo `@tanstack/react-virtual` usage; virtualization precedent
- `web/src/systems/*/lib/query-options.ts` per domain — the queries each domain view consumes
- `internal/api/core/extensions_dev.go:124-164` (stream epoch guard) + `session_catalog_stream.go` + `sse.go` — stream patterns for the patch stream
- `internal/api/core/cmd_palette*.go` + spec/routes files from task_01 — the family the view routes join
- `internal/codegen/openapits/` — generated TS for the vocabulary
- `web/e2e/fixtures/os-navigation.ts` — view selectors join the promoted palette helpers

### Dependent Files

- `internal/api/core/transport_parity_integration_test.go` — view route rows
- `openapi/compozy.json` + `web/src/generated/compozy-openapi.d.ts` — vocabulary/envelope types
- `web/src/systems/os/index.ts` + stories + MSW fixtures — new renderers and states
- `packages/site` palette docs page — view kinds section (owning page update)

### Competitor References

- `.resources/supercmd/src/renderer/src/raycast-api/list-runtime.tsx:258` — surgical virtualization pattern for capped lists/grids.

### Related ADRs

- [ADR-004](adrs/adr-004.md) — the four-kind vocabulary; adding a kind requires a new ADR
- [ADR-007](adrs/adr-007.md) — Go-owned typed vocabulary + JSON Patch + Adaptive-Cards versioning; host renders everything
- [ADR-002](adrs/adr-002.md) — extension content is data-only in the renderer (the contract these renderers uphold)
- [ADR-009](adrs/adr-009.md) — one vocabulary for both tiers (Tier-2 composes onto these renderers in task_08)

### Web/Docs Impact

- `web/`: shell generalization + three new kind renderers + all domain views under `web/src/systems/os/` (paths above), virtualization, stories, MSW view fixtures, e2e selector additions.
- `packages/site`: owning palette docs page gains the view-kind section; generated API reference for view routes.
- QA impact: flag only (walks in task_12): reset `docs/qa/scenarios/ET-palette-nested-views.md` and `ET-palette-sessions-view-switch.md` to `untested` (stack generalized to four kinds; sessions view re-registered — the standing `blocked-verify` on the latter resolves via the task_12 re-walk).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: the vocabulary + Tier-1 routes ARE the extension view contract (consumed by task_07's manifest family); no manifest change here — checked: `internal/extension` untouched this slice.
- Agent manageability: `GET /views/{id}` + `/views/{id}/stream` on both transports with structured errors; no new CLI verb (views are operator/render surfaces; agent enumeration ships via `list` from task_01).
- Config lifecycle: none — checked: no `config.toml` key touched; caps are named Go constants per Assumptions (deliberately not config).

## Deliverables

- Complete vocabulary + validation + caps + patch engine + generated TS with round-trip fixtures
- Four-kind shell generalization with unchanged stack semantics + unavailable/loading frames
- Curated built-in view for every list-bearing domain on the shared grammar/dictionaries
- Tier-1 view routes + fenced patch stream on both transports
- Visual Contract evidence bundles VC-01..12 **(REQUIRED)**
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] UT-040, UT-041, UT-042, UT-043, UT-044, UT-045, UT-046 — round-trip validate, wire caps/truncation, requires/fallback + telemetry, path-naming schema errors, unknown-kind honesty, patch fencing
- [x] UT-130, UT-131, UT-132, UT-133, UT-134, UT-135, UT-136, UT-137, UT-138, UT-139 — four kinds under one chrome, unavailable/slow frames, breadcrumb depth, domain template + tone parity, empty/cold states, detail pane, sanitized markdown rendering, form happy/error, submit-failure preservation + password, grid nav/media/empty
- [x] IT-005 — form-declared command submit → tool invocation with mapped args (seeded provider)
- [ ] E2E-008 — stack semantics across kinds (push/pop/Esc/reopen) — execution deferred to task_12
- [ ] E2E-009 — tasks domain view: truthful counts, honest empty, one-keystroke clear — implemented; real walk deferred to task_12
- [ ] E2E-010 — detail pane + action panel on entity rows (panel from task_04) — execution deferred to task_12
- [ ] E2E-011 — inline args + form-view variant preserving values on downstream failure — execution deferred to task_12
- [ ] E2E-012 — marketplace grid: 2D nav, placeholder, ⏎ parity — execution deferred to task_12

## Checkpoint

Implementation closed with a current full-gate record (`a3ecfb4aeb12bee995024f4d3f67900ac8b64056`). The loop exception keeps the flagged QA scenarios, E2E walks, and VC-01..12 capture in task_12; this checkpoint does not claim visual parity before those real walks run.

Compozy Impact Audit:

- Native tools: no impact — checked `internal/tools`, `internal/toolmeta`, native tool IDs, descriptors, schema digests, risk flags, and capability gates; the new read surfaces are HTTP/UDS view routes, not `compozy__*` tools.
- Extensibility and hooks: added the shared `ViewService`, frozen view vocabulary, read-only source enforcement, capability fallback, and Tier-1 HTTP/UDS routes. Manifest projection, extension lifecycle, MCP/bridge SDK exposure, and the official extension docs intentionally complete in task_07 against this contract; no hook event or config key changed here.
- Workspace data isolation: view descriptors are global definitions; opened payloads, revisions, streams, caches, and patch events are workspace-scoped. `workspace_id` is resolved canonically before service access on HTTP and UDS, and live parity tests prove no caller-supplied workspace can bypass that boundary.
- Official Compozy skill: no standalone update in this slice — `skills/compozy/` has no Tier-1 view operation until the extension manifest family exists; task_07 owns the combined extension palette entry so the skill never documents an unusable half-contract.

## Success Criteria

- Every assigned test case implemented and passing
- One rendering path: built-in and (later) extension payloads hit identical renderers — no parallel system (BR-21 structurally held)
- A patch can never apply out of order (property test + UT-046/SI-2)
- Every list-bearing domain reachable as a view with truthful counts and shared tone/attention dictionaries
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence
- `make gate` green including `make codegen-check` (vocabulary TS drift)
