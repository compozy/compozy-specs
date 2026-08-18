---
status: complete
title: Cutover and docs truth pass
type: refactor
complexity: high
---

# Task 5: Cutover and docs truth pass

## Overview

Executes ADR-003's hard cut as one change set: every Tauri delete target disappears (code, build targets, CI lanes, signing inputs, helper scripts, feed workflow, old channel objects), the docs are rewritten to the shipped shell (including the new release-operator runbook and the Chromium-first note), the official Compozy skill is completed, the announcement ships, and the reference-scan guard proves zero survivors. After this task, the repo is Go + TS with no bridge and no frozen artifact.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST execute every delete target enumerated in `_spec.md` Impact Analysis in this one change set: `desktop/src-tauri/**`; Rust/cargo CI lanes; `tauri-action` + `TAURI_*`/minisign secrets wiring; the listed `scripts/*.sh`; `.github/workflows/desktop-feed-repair.yml`; the old-format `releases.compozy.com` feed objects (ops follow-through recorded: bucket/DNS teardown); remaining Tauri-shaped code in `internal/desktoprelease`/`cmd/compozy-desktop-release` not already removed by task_04 — no alias, shim, or frozen artifact survives (SD-002).
2. MUST rewrite the desktop docs to the shipped shell: `getting-started/desktop-app.mdx` (bundled offline first run, boot ladder, update surfaces), `operations/desktop-app.mdx` (WebKitGTK troubleshooting deleted; new diagnostics truth), `getting-started/installation.mdx` (per-arch downloads), `getting-started/{index,web-ui}.mdx` touch-ups, `operations/support-bundles.mdx` consistency, `configuration/config-toml.mdx` consumer prose — plus the **new** `operations/desktop-release.mdx` runbook (publish/repair, known-good selection, `channel_cas_conflict` recovery, audit inspection).
3. MUST add the Chromium-first browser-support note (per US-028.AC-2, COPY.md claim standards) and verify every surviving desktop page against runtime truth.
4. MUST complete `skills/compozy/` for the migration surface: single update command + cancel, settings apply/cancel routes, app verbs unchanged, new statuses, shell facts.
5. MUST implement the UT-071 reference-scan guard (zero live references to deleted Tauri surfaces) and make it part of the suite, not a one-off.
6. MUST verify no `config.toml` setting is orphaned by the removal (Settings-affected rule): `[app]` keys remain live with the daemon consumer — evidence recorded.
7. MUST flag the QA tracker completely: every scenario touched by the program is either reset (`APP-*`, `REL-*` from earlier tasks) or newly added; charters referencing `tauri-driver` rewritten to the Playwright `_electron` reality; the announcement text (re-download + cleanup instructions per US-027.AC-3) delivered as release-notes content.
8. MUST close the workstream gates: `make gate-full` after the last mutation, with `make codegen-check` clean and boundaries green.
</requirements>

## Subtasks

- [x] 5.1 Delete sweep: `desktop/src-tauri/**` + scripts + workflows + secrets wiring + leftover Tauri code paths
- [x] 5.2 Old channel feed objects deleted; ops follow-through (bucket/DNS teardown) recorded in the completion notes
- [x] 5.3 Docs truth pass across the enumerated pages + new `operations/desktop-release.mdx` runbook
- [x] 5.4 Chromium-first note + COPY.md-conformant claims audit on touched pages
- [x] 5.5 `skills/compozy/` completion for the migration surface
- [x] 5.6 UT-071 reference-scan guard in the suite
- [x] 5.7 QA tracker completion: resets + new scenario files + charter rewrites (tauri-driver out)
- [x] 5.8 Announcement (release-notes content: re-download path, old-install cleanup)
- [x] 5.9 Settings-orphan evidence + final delete-target audit complete; the accepted loop plan owns the single `make gate-full` in Phase E

## Implementation Details

The authoritative delete list is `_spec.md` Impact Analysis "Delete targets"; the docs inventory is the Part I US-028 contract plus the Impact Analysis docs row. Docs follow `documentation-writer` + site conventions; product language follows `COPY.md` and `docs/_memory/glossary.md`.

Skills to activate: `documentation-writer`, `copywriting` (public pages/claims), `writing-skills` (skills/compozy edit), `eng-consolidate-test-suites` (UT-071 placement), `deslop` + `cy-final-verify` (workstream close).

### Relevant Files

- `desktop/src-tauri/**` — deleted wholesale (`desktop/schema/` stays)
- `scripts/{normalize-desktop-artifacts.sh,assert-desktop-signing-material.sh,verify-desktop-release-build-contract.sh,generate-desktop-update-key.sh,verify-desktop-signature.sh,publish-desktop-release.sh,repair-desktop-feed.sh}` + `.github/workflows/desktop-feed-repair.yml` — deleted (any not already removed by task_04)
- `.github/workflows/{ci.yml,release.yml}` — Rust/cargo/tauri-action remnants out
- `packages/site/content/docs/getting-started/{desktop-app.mdx,installation.mdx,index.mdx,web-ui.mdx}`, `operations/{desktop-app.mdx,index.mdx,support-bundles.mdx}`, `configuration/config-toml.mdx`, `cli/**` (generated refs current) — the truth pass
- `packages/site/content/docs/operations/desktop-release.mdx` — new runbook (N-012 owner)
- `skills/compozy/` — official skill completion
- `docs/qa/{scenarios,charters,journeys}/` + `state.csv` regeneration — tracker completion

### Dependent Files

- `Makefile` — desktop targets final shape (post task_03 re-point; cargo remnants out)
- `docs/_memory/` — no lesson/glossary change expected (verify: `capability`/naming untouched)
- `DESIGN.md`/`COPY.md` — untouched (verify no desktop-shell claims live there)

### Related ADRs

- [ADR-003](adrs/adr-003.md) — no bridge, delete-not-freeze, identity preserved
- [ADR-001](adrs/adr-001.md) — the WebKitGTK troubleshooting deletion rationale
- [ADR-007](adrs/adr-007.md) — retired origin (ops follow-through)

### Web/Docs Impact

- `web/`: none — checked; no web code changes (docs-only + deletion sweep).
- `packages/site`: the pages enumerated above (this task IS the docs impact owner for the program).
- QA impact: this task completes the program's flag set — verifies every reset/added scenario from tasks 01–04 exists with correct `qa_status: untested`, adds any missing (cutover behaviors: abandoned-install polling, announcement content), and rewrites the `tauri-driver` charters. The recorded walks happen in task_07.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: `skills/compozy/` completed (the one required surface); no other extension point touched — checked manifests/hooks/tools/registries/bridges/MCP.
- Agent manageability: docs for every agent path shipped (CLI refs current, runbook for the operator surface); no surface changes in this task.
- Config lifecycle: settings-orphan audit evidence — `[app]` keys live (daemon consumer), no key deleted/added; `config-toml.mdx` prose updated where consumer behavior is described.

## Deliverables

- One change set with every delete target executed and zero survivors (UT-071 green)
- Docs truthful end to end + new runbook + Chromium-first note
- `skills/compozy/` current; announcement content delivered
- QA tracker complete and consistent for the task_07 walk
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Success Criteria

- Every assigned test case implemented and passing
- `make gate-full` green (the workstream-close gate) with `make codegen-check` and boundaries clean
- Repo grep proves no live Tauri reference (UT-071 + manual evidence)
- Settings-affected audit recorded: no orphaned config key

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-071 — cutover reference-scan guard (zero live references to deleted surfaces)
