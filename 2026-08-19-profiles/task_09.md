---
status: pending
title: Extensions and Notification Presets per Profile
type: backend
complexity: high
---

# Task 9: Extensions and Notification Presets per Profile

## Overview

Delivers the extension capability: manifest `[[profiles]]` declarations + per-resource placement, per-profile enablement as the only enablement authority (routes + verbs), declared-profile auto-creation through the lifecycle protocol (create-once markers, seed snapshots, install summaries, durable needs-setup via `profile_credential_requirements`), per-profile extension secret bindings with the total env-binding resolution order, and notification-preset per-profile enablement — plus the S8/S9/S13 web surfaces, their Playwright journeys, and the extension-authoring docs/skill content.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

**Skills**: `eng-code-guidelines`, `golang-master`, `eng-contract-codegen-coship`, `eng-test-conventions`, `testing-boss`, `documentation-writer`; `eng-design`+`ui-craft`+`eng-ui-screenshot` for S8/S9/S13.

<requirements>
- MUST extend the manifest through the existing three-file chain (normalize/validate — never appended to `manifest.go`): `[[profiles]]` (name, color/icon, `[profiles.defaults]`, `[[profiles.credentials]]`) + per-resource `profile` placement key; invalid/reserved declared names or grammar-violating placements fail install (US-023.EC-5).
- MUST apply effective visibility = enabled(P) AND (unplaced OR placed-in-P's-name) at kit publish; placements naming an absent profile are dormant with the create hint (extension detail + install flow).
- MUST create declared profiles via `Manager.CreateDeclared` (never activates) inside the install/update pipeline, exactly once per (extension instance, profile name), recorded by durable markers; operator lifecycle always wins: no resurrection, existing name = bind never seed, updates never mutate, uninstall leaves the profile (ADR-002 complete).
- MUST persist the accepted declaration as the seed snapshot atomically with the lifecycle op (task_03's tables) so recovery reproduces the exact seed — never re-reading mutable extension state (IT-085 mutates the manifest post-crash to prove it).
- MUST make `profile_credential_requirements` the durable needs-setup authority: rows seeded from the snapshot in the apply transaction; *missing* = no matching profile vault secret; cleared only by the canonical vault write; survives restart/update/uninstall; removed on profile delete; `NeedsSetup`/`CredentialRequirement` projections on list/get across HTTP/UDS/CLI/Web.
- MUST ship enablement as exception rows only (absent = enabled) with the `_dx.md` routes and verbs (`compozy extension enable|disable [--profile]`), and prove the hard cut: old global columns gone, prior disabled state as `default`-profile exception rows, payloads expose only the per-profile authority (IT-081).
- MUST implement per-profile extension secret bindings over the rebuilt `extension_env_bindings` with the total resolution order `(profile,ws) → (profile,'') → ('',ws) → ('','')` — profile axis outranks workspace; rename leaves bindings untouched (id-keyed).
- MUST ship notification-preset enablement (same exception-row shape, uniform default-on) with routes, CLI verbs, Settings parity, and delivery routing that honors it.
- MUST complete the canonical event coverage (IT-079): every profile lifecycle path including `extension.profile_created`, `extension.enablement_changed`, `notification_preset.enablement_changed` — no standalone duplicate event test.
- MUST extend the IT-038 fixture with `profile_credential_requirements` rows (cumulative fixture contract in `_tasks.md`).
- MUST ship S8 (install/update summary naming declared profiles + credential needs; detail placement matrix; per-profile enablement toggle; needs-setup badge; dormant hints; uninstall copy), S9 (workspace "declares content for profiles X, Y — create?" hint with one action per name), S13 (per-profile preset rows on the notifications Settings surface).
- MUST ship the extension-authoring docs (manifest reference, placement/declared-profiles semantics) + official-skill updates in the same task.
</requirements>

## Visual Contract

Reference artboards under `docs/design/opendesign/profiles/` are produced by the design pass **before this task executes**; absent artboard at execution time → stop (blocked-precondition). Parity binds visual language only (SD-007/L-035 authorized deltas).

| ID    | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| ----- | -------------------------- | ----------------------------- | -------- | -------- | ---------------------------------- |
| VC-01 | `profiles-extension.html` — pre-install summary naming declared profiles + credential needs | extension install confirmation | 1440×900 | normative | manifest fixture → mock kit truth |
| VC-02 | `profiles-extension.html` — detail: placement matrix + per-profile enablement toggles | extension detail page | 1440×900 | normative | extension content → runtime truth |
| VC-03 | `profiles-extension.html` — needs-setup state on created profile | detail + switcher badge, unfilled asks | 1440×900 | normative | none |
| VC-04 | `profiles-extension.html` — dormant placement hint | detail with absent-profile placement | 1440×900 | normative | none |
| VC-05 | `profiles-hints.html` — workspace hint with 1..N names, one action per name | workspace registration/detail surface | 1440×900 | normative | repo names → fixture truth |
| VC-06 | `profiles-hints.html` — partially adopted state (hint dropped adopted name) | workspace surface after one adoption | 1440×900 | normative | none |
| VC-07 | `profiles-settings.html` — per-profile preset enablement rows | notifications Settings, active profile | 1440×900 | normative | preset names → runtime truth |

Evidence per row: `.compozy/tasks/profiles/evidence/visual/task_09/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [ ] 9.1 Manifest chain: `[[profiles]]` + placement key parsing, normalization, validation (reserved/grammar/undeclared-name rules).
- [ ] 9.2 Kit-publish placement filter (enablement AND placement) + dormant-placement hint payloads.
- [ ] 9.3 Install/update pipeline: declared-profile plan → summary → `CreateDeclared` with markers + seed snapshot; uninstall marker removal; update-only-new-declarations.
- [ ] 9.4 `profile_credential_requirements` population + canonical-vault-write clearing + `NeedsSetup`/`CredentialRequirement` projections everywhere.
- [ ] 9.5 Enablement routes/verbs + exception-row store + IT-081 hard-cut proof; preset enablement routes/verbs + delivery routing.
- [ ] 9.6 Env-binding resolution (total order) + per-profile secret binding overrides.
- [ ] 9.7 Event coverage completion (IT-079) over the canonical observability suite.
- [ ] 9.8 Web S8/S9/S13 composites (`ExtensionDeclaredProfiles`, `WorkspaceProfilesHint`, preset rows) + Playwright journeys + visual evidence.
- [ ] 9.9 Docs: extension-authoring manifest reference + profiles interplay; skill updates; QA scenario flags; IT-038 fixture extension.

## Implementation Details

The install pipeline consumes task_03's `CreateDeclared` + snapshot machinery; this task owns manifest → plan → summary → creation → needs-setup. Mock kits drive E2E-008 (`acpmock`/`extensiontest` harness).

### Relevant Files

- `internal/extension/manifest.go:88-130` + `manifest_normalize.go` + `manifest_validate.go` + `extension_validation.go` — the three-file chain.
- `internal/extension/install_managed.go:31-60` — managed install root + containment (marker/creation hook point).
- `internal/extension/capability_resource_policy.go` + `surfaces/registry.go` — placement flows through the task_08 lattice.
- `internal/extension/manager_bridge_delivery.go` + `internal/extensiontest/` — harness for install/enablement integration fixtures.
- `internal/store/globaldb/schema/definitions/33_extensions.sql` — enablement/marker/env-binding tables (landed in task_02).
- `internal/notifications/` preset store + delivery routing; `37_notifications.sql` preset tables.
- `internal/api/spec/registry_extensions.go` — route registry to extend (+ notifications presets).
- `web/src/systems/extensions/` + `web/src/systems/notifications/` + `web/src/systems/workspace/` — S8/S13/S9 surfaces.
- `skills/compozy/references/extension-authoring.md` + `references/extensions.md` — docs targets.

### Dependent Files

- `internal/cli/` extension + notification-preset command files — `--profile` variants and effective-state listings.
- `web/src/generated/compozy-openapi.d.ts` — regen for enablement/needs-setup payloads.
- `internal/vault/extension_refs.go` — per-profile extension secret refs (grammar from task_08).

### Competitor References

- `.resources/hermes/hermes_cli/profiles.py:1-58` — distribution-owned vs user-owned line ("nothing is copied") the marker semantics preserve.

### Related ADRs

- [ADR-002](adrs/adr-002.md) — declared profiles: create-once, bind-never-seed, operator wins.
- [ADR-006](adrs/adr-006.md) — name-binding shared with the repo layer (dormancy/hints).
- [ADR-009](adrs/adr-009.md) — credential asks → vault-only fulfillment.
- [ADR-010](adrs/adr-010.md) — enablement is organization, never permission vocabulary.

## Deliverables

- Manifest contract validated at build/install; placement + enablement effective everywhere surfaces project.
- Declared profiles created once, seeded from snapshots, marker-protected; needs-setup durable and clearable only by the canonical vault write.
- Enablement + preset-enablement routes/verbs/UI with the old global authorities gone.
- S8/S9/S13 live with passing visual evidence.
- Extension-authoring docs + skill updates.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [ ] UT-053..UT-062 — manifest validation/normalize; visibility truth table; dormant placement; install plan/summary; bind-not-seed; marker semantics; enablement exception rows; preset default-on.
- [ ] UT-066 — secret binding resolution (user default, profile override scoped).
- [ ] UT-090 — preset enablement resolution (notifications suite).
- [ ] IT-049..IT-052 — declared-profile install/bind/update/uninstall + needs-setup durability + never-resurrect.
- [ ] IT-053..IT-056 — placement matrix live; placement updates; dev-link composition + secret overrides; preset delivery routing.
- [ ] IT-071 — preset enablement parity (CLI/routes/Settings on one store).
- [ ] IT-079 — canonical event coverage completeness.
- [ ] IT-081 — enablement hard cut proven end to end.
- [ ] IT-082 — env-binding rebuild: total resolution order + cleanup triggers + rename stability.
- [ ] IT-085 — declared-seed crash barrier; requirement rows outlive op retention.
- [ ] E2E-008 — install with declared profile (mock kit): summary → creation → needs-setup → secret set completes.
- [ ] E2E-022 — S9 repo hint → adopt → bind → hint drops.
- [ ] E2E-023 — S8 placement matrix, enablement toggle, needs-setup badge, dormant hint.
- [ ] E2E-026 — S13 preset rows per active profile, persistence, per-profile state.

### Web/Docs Impact

- `web/`: `web/src/systems/extensions/` (install summary, detail matrix, toggles, badges), `web/src/systems/workspace/` (hint), `web/src/systems/notifications/` (preset rows), generated types regen.
- `packages/site`: extension-authoring manifest reference (+ `[[profiles]]`/placement), extensions docs page (per-profile enablement), notifications docs (preset enablement); generated CLI docs regen (`extension`, `notifications`).
- QA impact: new scenarios — add content-addressed untested files for declared-profile install journey, per-profile enablement, dormant placements, and preset enablement; reset existing extension enable/disable scenarios (authority moved per-profile). Walk owned by task_13.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: this task IS the plan — manifest `[[profiles]]` + placement key, kit-publish filter, capability grants via the task_08 lattice, per-profile secret bindings, MCP sidecar entries per profile, extension-authoring/protocol docs.
- Agent manageability: enablement + preset verbs/routes with structured output and parity (IT-071); needs-setup discoverable via `profile list`/`GET /api/profiles/{name}`; deterministic errors on invalid manifests.
- Config lifecycle: no new keys; manifest schema documented; `ExtensionsConfig.Resources.MaxScope` value docs updated with the lattice values (change landed in task_08).
