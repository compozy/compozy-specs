---
status: completed
title: Release pipeline and channel authority
type: infra
complexity: high
---

# Task 4: Release pipeline and channel authority

## Overview

Replaces the Tauri release machinery with the Electron pipeline and the linearized channel authority: per-arch builds with signing/notarization, merged mac manifests, the `channel-beta` git branch read by the updater's generic provider, `compozy-desktop-release publish|repair` with ref-CAS semantics and structured output, `compat.json` production with the publication invariant, the new artifact inventory, and the packaged-app smoke — all wired into CI in lockstep with the existing goreleaser train.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST build per-arch macOS artifacts (arm64 + x64 dmg/zip) and Linux x64 (AppImage + deb) via electron-builder with the static config + minimal channel/signing generator (ADR-008); notarytool + staple for macOS; the mac update-zip symlink guard from the reference pipeline.
2. MUST implement the channel authority per Part II "Publish and repair authority": immutable versioned releases for payloads; channel manifests committed to the `channel-beta` branch; **publish = payload upload+verify → one commit (all platform manifests) → ref update with expected-old-SHA CAS**; repair = inventory-verified CAS to a known-good commit; audit = operation-id-keyed commits; deterministic `channel_cas_conflict` on lost races; idempotent re-runs (ADR-007 note, B-024).
3. MUST emit the structured JSON operator contract for both subcommands (`{operation, channel_ref_before/after, verified_inventory, audit_commit, outcome}` + error codes `inventory_incomplete`, `channel_cas_conflict`, `verification_failed`).
4. MUST produce `compat.json` (`{runtime_version, min_app_version}`) listed in `checksums.txt`, and enforce the **publication invariant**: refuse publishing a release whose `min_app_version` exceeds the previous channel generation's app version (B-020).
5. MUST rewrite the artifact inventory to the per-arch exact set and delete the Tauri-shaped `compozy-desktop-release` subcommands (`channel-config`, `assert-comparator`) in the same change.
6. MUST keep the release-asset ordering invariant (payloads verified before the manifest flip) and the app-track feed consumable by the updater's **generic provider** over the raw branch URLs — rehearsed against the real provider read path, not only mocks.
7. MUST adapt the packaged-app smoke (isolated home, boot to `/api/status`, teardown-clean) to the Electron artifacts and gate publication on it, preserving the preflight secret assertions and provenance checks.
8. MUST wire CI: desktop build lanes (two mac builders + linux) in `release.yml` lockstep with goreleaser, `desktop-channel` concurrency group as defense-in-depth, mock-update-server fixtures for E2E-031, N→N+1 rehearsal.
</requirements>

## Subtasks

- [x] 4.1 electron-builder config (static + generator): per-arch mac, linux targets, identity frozen (`com.compozy.os`, `compozyos://` protocol), bundled-runtime extraResources per arch
- [x] 4.2 Signing/notarization lane (notarytool/staple; mac update-zip symlink guard; unsigned local path preserved for task_03 e2e)
- [x] 4.3 Mac manifest merge + per-arch differential settings (cross-arch guard)
- [x] 4.4 `compozy-desktop-release publish|repair` (ref-CAS authority, audit commits, structured output, known-good selection) + deletion of `channel-config`/`assert-comparator`
- [x] 4.5 `compat.json` production + publication invariant in `publish`
- [x] 4.6 Inventory rewrite (`internal/desktoprelease`) to the per-arch exact set
- [x] 4.7 Smoke successor for Electron artifacts + provenance assertions
- [x] 4.8 CI lanes: build matrix, channel bootstrap (`channel-beta` branch), concurrency group, mock feed server + release-smoke fixtures, N→N+1 rehearsal (E2E-031)
- [x] 4.9 Full assigned suite green; rehearsals recorded

## Implementation Details

`_spec.md` Part II: Publish and repair authority, Release compatibility gate, Integration Points, Impact Analysis (pipeline rows). The CLI/runtime goreleaser lanes are untouched — this task adds the app lanes beside them in the same tag-driven train (lockstep versioning).

Skills to activate: `golang-master`, `eng-code-guidelines` (authority code), `eng-test-conventions`, `testing-boss`, `eng-consolidate-test-suites`, `context7` (electron-builder/electron-updater provider docs).

### Relevant Files

- `internal/desktoprelease/{model.go,policy.go,manifest.go,inventory.go,channel_config.go,schema_heads.go,canonical_json.go,release_test.go}` — the package this task reshapes (channel_config + schema_heads die; inventory/policy rewritten; authority added)
- `cmd/compozy-desktop-release/main.go` — subcommand surface (add publish/repair, delete channel-config/assert-comparator)
- `.github/workflows/release.yml` — desktop lanes (`desktop-dry-run`, `desktop-build`, `desktop-publish`, `desktop-smoke`, `desktop-finalize`) to rewrite; goreleaser train integration points
- `.github/workflows/ci.yml:342` — desktop job re-point
- `scripts/{normalize-desktop-artifacts.sh,assert-desktop-signing-material.sh,verify-desktop-release-build-contract.sh,prepare-desktop-smoke-artifact.sh,smoke-desktop-release-artifact.sh,assert-desktop-artifacts.sh,build-desktop-feed.sh,assert-desktop-version-strictly-greater.sh}` — Tauri-coupled scripts replaced or deleted here (final sweep in task_05)
- `.goreleaser.yml` — runtime archive train (unchanged; lockstep reference)
- `internal/update/types.go` — feed/asset naming contracts the authority must stay consistent with

### Dependent Files

- `desktop/` builder config from task_03 — this task signs/publishes what task_03 builds
- `docs/qa/scenarios/REL-*.md` — release-contract scenarios affected (flagged below)

### Related ADRs

- [ADR-007](adrs/adr-007.md) — GitHub Releases + channel-branch/ref-CAS note
- [ADR-008](adrs/adr-008.md) — per-arch mac artifacts
- [ADR-003](adrs/adr-003.md) — old feed deletion context (executed in task_05)
- [ADR-009](adrs/adr-009.md) — operation semantics the rehearsals exercise

### `.resources/` References

- `.resources/synara/scripts/lib/{mac-update-zip.ts,mac-dmg-finalize.ts}` + `scripts/merge-mac-update-manifests.ts` — mac zip symlink guard, notarize/staple, manifest merge
- `.resources/t3code/scripts/{merge-update-manifests.ts,mock-update-server.ts,release-smoke.ts}` — merge + mock-feed + rehearsal fixture shapes
- `.resources/hermes/apps/desktop/scripts/notarize.mjs` — notarytool credential modes
- `.resources/synara/.github/workflows/release.yml` — payloads-before-manifests ordering, packaged-startup verification, provenance assertions

### Web/Docs Impact

- `web/`: none — checked; no web surface consumes release infrastructure.
- `packages/site`: none in this task — the release-operator runbook (`operations/desktop-release.mdx`) and page updates land in task_05's truth pass against the finished authority.
- QA impact: reset to `untested` — `docs/qa/scenarios/REL-release-archive-update-contract.md` and `REL-beta-channel-contract.md` (publish path replaced); add content-addressed `untested` scenario for channel repair (`REL`-class, ref-CAS known-good restore). Walk in task_07.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked; release tooling exposes no extension surface.
- Agent manageability: `compozy-desktop-release publish|repair -o json` structured operator contract with deterministic error codes (release-operator surface, listed in the spec's Agent Manageability Plan).
- Config lifecycle: none — no `config.toml` keys; channel/signing inputs are CI secrets + generator flags (enumerated in the workflow).

## Deliverables

- Signed, notarized per-arch artifacts publishing to a versioned GitHub Release in the tag train
- `channel-beta` branch live; publish/repair authority operational with audit commits
- `compat.json` in the release with the publication invariant enforced
- Smoke + rehearsals (interrupted publish, repair, N→N+1) recorded in CI
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md`, the test contract — read each ID's full definition there before writing tests.

- [x] UT-069–UT-070 — per-arch inventory exact-set + version monotonicity policy
- [x] UT-081 — authority: known-good selection, missing-asset refusal, CAS-after-verification ordering, `channel_cas_conflict`, idempotent re-run
- [x] IT-023 — compatibility-bump protocol (publisher gate + two-release flow)
- [x] E2E-029–E2E-033 — packaged smoke, manifest-merge routing, N→N+1 rehearsal, publish-authority ordering (interrupted), repair rehearsal on the real provider read path

## Success Criteria

- Every assigned test case implemented and passing
- A full tag-driven dry run produces the exact inventory, passes smoke, and flips the channel exactly once
- Interrupted-publish rehearsal leaves the channel on one complete generation at every interruption point
- `make gate` green; no Tauri-shaped subcommand or script referenced by the new lanes
