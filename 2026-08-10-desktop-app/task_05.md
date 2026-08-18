---
status: completed
title: "Release pipeline: build matrix, signing, feeds, gates, custody runbook"
type: infra
complexity: critical
---

# Task 5: Release pipeline: build matrix, signing, feeds, gates, custody runbook

## Overview

Ships the desktop release lane: the pinned 3-job build matrix with full signing (Apple notarization + DMG stapling, Azure Artifact Signing inside the bundler, minisign updater artifacts), the fail-closed publish job implementing all 12 integrity gates with the payloads-first/manifest-last atomic feed publish to `releases.compozy.com`, the signed `runtime.json` generation, and the key-custody runbook + launch-gate checklist that ADR-004 mandates before the first public release.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. Build matrix MUST be pinned: `macos-15` building `universal-apple-darwin` (single job, both rust targets, universal fan-out to both `darwin-*` feed keys), `ubuntu-22.04` (webkit2gtk-4.1/glibc floor), `windows-latest`; `fail-fast: false`; `tauri-action@v1` (renamed inputs; `releaseDraft: true`, `uploadUpdaterJson: false`).
2. Signing MUST fail closed: assert presence of ALL signing material before building (an unsigned build otherwise SUCCEEDS silently); macOS via ASC API key (env remap from legacy org secret names — never the Apple-ID path); Windows via Azure **Artifact Signing** `artifact-signing-cli` through object-form `signCommand` INSIDE the bundler (post-build signing invalidates minisign); NSIS-only, `currentUser` + updater `passive`.
3. MUST notarize + staple the DMG explicitly (tauri#7533 — Tauri never does) and gate on `xcrun stapler validate` exit 0 for BOTH `.app` and `.dmg`; never `--skip-stapling`.
4. Publish job MUST implement the 12 gates from analysis 07 §7 (all-green barrier, exact inventory, minisign verify with the SHIPPED pubkey, staple validation, version-string match, feed schema validation, payloads-first→manifest-last single PutObject, post-publish re-fetch + payload re-verify, GitHub draft published LAST, default-comparator assertion, strictly-greater-than-live-feed assertion) plus the reserved-`stable/` publication guard.
5. Feed layout MUST match the TechSpec: `desktop/<channel>/latest.json` + `desktop/<channel>/runtime.json`+`.minisig` (no-cache) · payloads under `desktop/v/<version>/…` and `desktop/v/runtime/<version>/…` (immutable). `runtime.json` is generated from the existing release-plan channel outputs and signed with the update-feed key; `schema_heads` included. GitHub Releases stays the archive, never referenced by the feed.
6. Channel is compiled in via the CI-generated `tauri.channel.conf.json` (beta only; `stable/` reserved and unpublished).
7. MUST write `docs/runbooks/update-signing-key.md` with the full ADR-004-mandated contents (key identity, custody roster, restore drill, rotation procedure with the transitional table + stranded-tail statement, loss procedure, compromise procedure, blast radius) and the launch-gate checklist with owners/lead times (analysis 07).
8. Keypair generation happens OFF-CI with a password (never with `CI` set); org secrets are copies, not backups.
9. E2E-019 (draft-release rehearsal incl. simulated one-platform failure → no feed) MUST run before this task closes.
</requirements>

## Subtasks

- [x] 5.1 `release.yml` desktop build jobs: matrix, pins, deps install, rust-cache, signing-material assertions, env remaps
- [x] 5.2 Channel config generation + `tauri-action@v1` invocation + workflow artifacts
- [x] 5.3 DMG notarize+staple post-step + stapler-validate gates
- [x] 5.4 Publish job: barrier, inventory + verify-sig scripts (base64+minisign), feed assembly + schema validation, R2 uploads (payloads→manifest), post-publish verification, draft publication
- [x] 5.5 `runtime.json`+`.minisig` generation from release-plan outputs (version, platforms, sha256, schema_heads)
- [x] 5.6 BR-10 gates: default-comparator assertion + strictly-greater-than-live-feed check + `stable/` guard
- [x] 5.7 Custody runbook + launch-gate checklist docs; validate gate scripts locally (UT-083–086, UT-116 as script-level tests)
- [ ] 5.8 E2E-019 rehearsal: draft release end-to-end incl. forced single-platform failure — execution is assigned to Task 07 by the loop directive that reserves all QA for Tasks 06–07.

## Implementation Details

Pipeline shape, gate list, script sketches, and production patterns: `analysis/07_analysis_tauri-dist-release.md` (§§1–5, 7, pipeline part 5/5) — follow it closely; it encodes verified 2026 tool names and live-bug workarounds. Existing release conventions: `.github/workflows/release.yml` (channel plan outputs `RELEASE_CHANNEL`/`NPM_TAG` at :342-343/:411-425/:603-606), `.goreleaser.yml` (matrix, cosign, archives — untouched). Org credentials + R2 per ADR-004. Test note: UT-083–086/UT-116 are script-level validations runnable locally against fixtures (not full CI runs).

### Relevant Files

- `.github/workflows/release.yml` — desktop jobs join the existing tag/channel flow
- `scripts/` (repo conventions) — verify-sig, assert-artifacts, build-feed, assert-version-strictly-greater, notarize-staple-dmg
- `docs/runbooks/update-signing-key.md` — new
- `desktop/src-tauri/tauri.conf.json` — bundle targets, `createUpdaterArtifacts`, signCommand (windows conf)

### Dependent Files

- `desktop/` crate (tasks 01/03) — the thing being built; updater endpoint + pubkey embedded
- `releases.compozy.com` R2 bucket + cache rules (admin gate)

### Related ADRs

- [ADR-004](adrs/adr-004.md) — signing reuse, own-domain feed, custody gate (this task's contract)
- [ADR-005](adrs/adr-005.md) — channel layout, stable reserved
- [ADR-011](adrs/adr-011.md) — signed runtime publication (generation side)
- [ADR-012](adrs/adr-012.md) — release display names

## Deliverables

- Green draft-release run producing signed/notarized/stapled artifacts for all 3 platforms + published beta feed on R2 passing post-publish verification
- Custody runbook + launch-gate checklist reviewed and committed
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [x] UT-083–UT-086 — feed/manifest schema gates, signature presence, stable-path guard
- [x] UT-116 — BR-10 regression gates (default comparator + strict monotonicity)
- [ ] E2E-019 — release-integrity rehearsal (dry-run without secrets hard-fails; draft publish + verify; single-platform failure → no feed) — retained as a Task 07 QA obligation.

## Web/Docs Impact

`docs/runbooks/` new runbook; download-page prerequisites were shipped in task_04 (verify links). No `web/` change. **QA impact:** flag only — release/update behavior scenarios planned in task_06, walked in task_07 (update rehearsal charter).

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — CI + docs only. Checked: no runtime contract, manifest, or registry change.
- Agent manageability: none new — the pipeline publishes the feeds the shell/CLI consume.
- Config lifecycle: none — checked: no config key; channel stays build identity (ADR-010/012).

## References

- `analysis/07_analysis_tauri-dist-release.md` (primary — pipeline, gates, custody, launch checklist)
- `analysis/01_analysis_tauri-legacy.md` (legacy CI gates as historical reference only; tool names are stale)
- `.github/workflows/release.yml`, `.goreleaser.yml` (existing channel plan + signing conventions)

## Success Criteria

- Every assigned test case implemented and passing; E2E-019 rehearsal evidence recorded
- A tag push produces a fully-signed draft release; removing any signing secret fails the build before artifacts exist
- Feed on R2 passes schema + signature + monotonicity gates; `stable/` remains empty and guarded
