---
status: completed
title: Prepare the v0.3 beta cut — release identity, channel mechanics, single-cut runbook
type: infra
complexity: high
---

# Task 10: Prepare the v0.3 beta cut — release identity, channel mechanics, single-cut runbook

## Overview

Prepares everything in-repo for the single external cut without executing it: release identity moves to the legacy channel names; a `workflow_dispatch` release path delegates deterministic policy validation to the pinned `github.com/compozy/releasepr@v0.0.24` planner; the front-door README documents beta installation and points to the deprecated legacy branch; the hosted installer serves one contract; and one runbook covers the entire cut — `legacy/v0.2` branch, squash merge of `v0.3` into `main`, beta publish, `compozy.com` pointing, `compozy/agh` archival, and live checks. Task 13 supplies fresh local and release-PR dry-run evidence. After it is green, Pedro executes this task's recorded single-cut runbook, including the live installer, registry, and cosign checks. Per brief round-11, nothing this task adds is removed by a later task.

<critical>
- ALWAYS READ `_brief.md` (round 11 governs staging), `_techspec.md`, `_tests.md`, `_tasks.md`, and the applicable ADRs before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — do not add compatibility aliases, dual identity paths, dual-contract installers, silent channel fallbacks, temporary artifacts scheduled for later removal, or unverifiable publish shortcuts
</critical>

<requirements>
- MUST flip release identity to the active legacy channel names: binary `compozy`, brew formula `compozy` in the existing tap, npm package `@compozy/cli`, and nfpm packages `compozy`. The legacy GoReleaser AUR block is commented out and no active AUR channel exists; v0.3.0 MUST NOT create or claim a `compozy-bin` publication.
- MUST provide an explicit `workflow_dispatch` release path with release ref, unprefixed SemVer version such as `0.3.0-beta.1`, and `beta|stable|legacy` channel inputs. Pin `PR_RELEASE_MODULE` and the vendored release skill to `github.com/compozy/releasepr@v0.0.24`, invoke `pr-release plan` once for those explicit inputs, and consume its `release_ref`, `release_commit`, `release_version`, `release_tag`, `release_channel`, `github_prerelease`, `github_make_latest`, `npm_tag`, and `homebrew_skip_upload` outputs without re-derivation. The planner MUST resolve the supplied ref to checked-out `HEAD`, reject a version beginning with `v`, and fail closed if `v${version}` already exists locally or on `origin`; no branch, pull-request ref, git-cliff history, or mixed repository lineage may infer version or channel. The Compozy workflow—not the read-only planner—MUST create the annotated tag and perform publication. The beta outputs MUST create a GitHub prerelease, publish `@compozy/cli` only with npm dist-tag `beta`, leave npm `latest` untouched, and skip the Homebrew upload/bump. The `beta` tag mechanism is concrete and tested: use a verified GoReleaser npm option when supported or an explicit checked `npm dist-tag add`, never an assumption based on prerelease SemVer. The `stable` channel is the post-beta promotion path executed outside this spec; the workflow ships it working, this migration never runs it.
- MUST support an emergency `legacy/v0.2` maintenance release from that branch without moving `@compozy/cli` or the `compozy` formula to the v0.3 identity.
- MUST make the hosted installer serve exactly one contract (brief round-11): `packages/site/public/install.sh` implements only the v0.3 Sigstore verification contract and installs the documented beta version explicitly — it never resolves to, validates, or falls back to a legacy PEM/SIG release. During the beta window the script's default target is the documented `v0.3.0-beta.N` (explicit pin or verified prerelease resolution — a tested mechanism, not an assumption); when 0.3.0 final ships, the default becomes latest stable through normal release maintenance outside this spec. Legacy installs belong to `legacy/v0.2` collateral and are never this script's job.
- MUST use the destination root entrypoint as the canonical Go install target: `go install github.com/compozy/compozy@<version>`. The current root `main.go` already owns this contract, so no `/cmd/compozy` alias or user decision is needed. Prove the renamed root target in a clean environment. `go install @latest` resolves the last stable tag (v0.2.x) until the post-beta stable release, so the README and site document the explicit `@v0.3.0-beta.N` form as the beta install path.
- MUST repin the release repository slug and the cosign certificate identity regexp in the same change as the workflow rename, and MUST keep the workflow's installer contract checks passing.
- MUST update self-update detection and instruction strings to the legacy channel identities, and make a beta build track the beta channel: the update check offers newer v0.3 prereleases and never the v0.2.x stable line (no downgrade), and instruction strings point at the beta install paths (`npm install -g @compozy/cli@beta`, the hosted installer). Homebrew upgrade instructions are emitted only when the formula actually ships the running line (0.3.0 stable, post-beta).
- MUST verify the destination's already-MIT `LICENSE` and release metadata after the Stage-0 copy, correcting only stale `BSL-1.1` values that actually survive in a published manifest. There is no relicensing and no requirement to manufacture edits to already-correct package manifests (ADR-005, amended).
- MUST author the front-door README as the complete beta front door: a full beta installation section (npm `@compozy/cli@beta`, hosted `install.sh`, `go install …@v0.3.0-beta.N`, source build) consistent with the site's install copy — Homebrew is omitted from the install section until the 0.3.0 stable bump restores it, at most a short note that Brew returns at stable; an explicit pointer to `legacy/v0.2` as the deprecated previous version with its maintenance policy; a complete inventory of legacy inbound heading anchors with the badge row for SEO — preserve or explicitly map each anchor that has a semantic v0.3 successor by placing it at that successor, no empty heading stubs, and a migration-guide `no successor` row for removed sections; an OS-first body in the people-first register; and repointed repository-scoped widgets. Beta status is ordinary copy that the post-beta stable release updates — never a banner or temporary artifact owned here and removed elsewhere.
- MUST author (not execute) the single-cut runbook covering, in order: pre-cut gate (Task 13 green); `legacy/v0.2` branch creation from `main` with its maintenance-notice commit; squash merge of `v0.3` into `main`; tag and publish `v0.3.0-beta.1` through the pinned workflow (`channel=beta`); pointing `compozy.com` at the `packages/site` deployment and retiring both the legacy marketing deployment and `agh.network` (DNS/hosting owner executes; no redirect infrastructure); repository metadata (description, `--homepage https://compozy.com`, topics, social preview); npm collateral; the Homebrew tap deprecation notice that hides/invalidates Brew until 0.3.0 stable (the legacy formula is marked deprecated, pointing at the beta install paths); the terminal `compozy/agh` archival commit (pointer README plus `go.mod` deprecation notice) before archive; `@compozy/agh` npm deprecation after its terminal publish; CodeQL parity; and the post-publish live checks. Also author the separate emergency `legacy/v0.2` maintenance-release runbook.
- MUST extend the existing `packages/site/lib/__tests__/public-install-contract.test.ts` owner to cover root README, release header/footer templates, and `cliff.toml` release identity; do not add a duplicate static contract suite.
- MUST make the runbook deterministic for the remaining non-product checks: audit legacy secret history and rotate any exposed credential; verify/provision `RELEASE_TOKEN`, `GORELEASER_KEY`, `NPM_TOKEN`, the renamed `COMPOZY_WEB_ASSETS_TOKEN` when the external web-assets path is authorized, and `DAYTONA_API_KEY`, `DAYTONA_API_URL`, `DAYTONA_ORGANIZATION_ID`, `DAYTONA_SNAPSHOT`, `DAYTONA_IMAGE`, `DAYTONA_SSH_HOST`; preserve the legacy `security-extended` CodeQL coverage without importing unrelated `auto-docs.yml`/`claude.yml`; and, after the terminal legacy publish, deprecate every published `@compozy/agh` version with a verified pointer to `@compozy/cli`.
- MUST create the destination root `imgs/` front-door collateral for the README. That directory disappears in the Stage-0 replacement and is created from current `docs/design/screen.png` plus legacy composition references; this is new collateral, not regeneration of files present in the destination. Task 11 owns the site landing hero media.
- MUST NOT execute any irreversible external action (publish, branch creation on the remote, squash merge, DNS change, archive). Task 13 green gates Pedro's single-cut runbook execution; the recorded post-publish installer, registry, and cosign checks occur after publication and are not preconditions for Task 13 itself.
</requirements>

## Subtasks

- [x] 10.1 Flip goreleaser identity: project, binary, brew, npm, nfpm, homepage, description, ldflags path
- [x] 10.2 Pin `releasepr v0.0.24` in the workflow and vendored skill; configure dispatchable beta/stable/legacy paths that consume one authoritative plan, retain workflow-owned annotated tag creation, and apply the emitted GitHub/npm/Homebrew policy without branch or version inference
- [x] 10.3 Repin the release repository slug and cosign identity; keep installer contract checks green
- [x] 10.4 Rework `install.sh` to the single v0.3 Sigstore contract with an explicit, tested beta version target; preserve Task 02's `COMPOZY_*` namespace; prove the canonical root Go-install target in a clean environment
- [x] 10.5 Update self-update detection paths and upgrade instruction strings; make beta builds track the beta channel with no v0.2.x downgrade and no Homebrew instructions until 0.3.0 stable
- [x] 10.6 Verify continuous MIT identity across `LICENSE` and published manifests; correct only stale distribution metadata that survives the copy
- [x] 10.7 Author the front-door README: beta install section, `legacy/v0.2` deprecation pointer, explicit legacy-anchor disposition map, preserved semantic anchors, badges, and OS-first body
- [x] 10.8 Create destination root `imgs/` front-door collateral from the current OS UI and legacy visual references: OS-shell capture plus a loop/runtime diagram; the site landing hero media stays with Task 11
- [x] 10.9 Author the single-cut runbook (legacy branch → squash merge → beta publish → `compozy.com` pointing and `agh.network` retirement → archival → collateral → live checks) and the emergency legacy-maintenance runbook; separate pre-publish evidence from the post-publish live-check list
- [x] 10.10 Verify community files and CodeQL parity against the legacy repository configuration
- [x] 10.11 Implement IT-017, then run `make installer-check`, `make codegen-check`, `goreleaser check`, and `make verify`

## Implementation Details

Follow ADR-005 (as amended by brief round-11) for distribution identity, `_content-plan.md` §C for channel collateral, `_brief.md` round-11 for single-cut staging, `_brief.md` round-10 for the pinned release-planning contract, and TechSpec §Development Sequencing steps 5-6.

The heavy release checks live in the release-PR dry-run lane, never on a schedule (repo CI policy). The runbook is an authored artifact in this task; its execution is a human step gated on Task 13's fresh pre-publish evidence. `pr-release plan` is a read-only validation boundary: it checks explicit inputs, checked-out commit identity, and local/remote tag absence, then emits authoritative policy. The repository workflow still creates the annotated tag and publishes. It must not infer a channel from a branch name, recalculate planner outputs, or silently fall through to a stable publish.

The post-beta stable release (`channel=stable`: move npm `latest`, bump Homebrew, re-tag beta copy to stable) is normal release work outside this spec. This task ships the workflow capable of it and the runbook may reference it as a closing note, but no spec task executes or gates it.

### Relevant Files

- `.goreleaser.yml` — project name, metadata, build, ldflags, archive, brews, nfpms, npms, cosign, release repo, extra files, and confirmation that no active AUR publisher exists
- `.goreleaser.release-header.md.tmpl`, `.goreleaser.release-footer.md.tmpl`, and `cliff.toml` — shipped release identity, links, and changelog repository references
- `.github/workflows/release.yml:1-230,287-470` — explicit ref/version/channel inputs, pinned `PR_RELEASE_MODULE`, `pr-release plan` output consumption, workflow-owned annotated tag creation, npm authentication and tag mechanism, release creation, secrets, pinned cosign version, publish paths, and all installer contract greps for beta/stable/legacy
- `.agents/skills/releasepr/**` — vendored `v0.0.24` release planning contract, workflow guidance, source manifest, and upstream pins; no branch-based version/channel inference remains
- `packages/site/public/install.sh:4,7,9-10,17,26-27,39,194,214,248` — release repo, cosign identity regexp, environment variables, docs URLs, archive/target/tmp names, and the single-contract beta version target
- `internal/update/types.go`, `internal/update/detect.go:132,136` — release API/repository/module/cosign identity plus npm and scoop install-path probes
- `internal/update/manager.go:93,127,165` + `github.go:154` — home requirement message, release wording, temp dir, user agent
- `internal/cli/lifecycle.go` + `lifecycle_test.go:81,132,137,281-282` — managed-lifecycle upgrade strings and binary name
- `README.md` (root) — becomes the beta front door; anchors, badge row, install section, and the `legacy/v0.2` pointer are load-bearing for inbound links and upgrade guidance
- `docs/design/screen.png`, new destination `imgs/`, and legacy `../compozy/imgs/**` — source capture, new front-door collateral, and visual references; retired TUI/pipeline assets are not copied as current truth
- `the CompozyOS repo` (branch `main`) — legacy README anchors, badges, star-history and contributor widgets, `.github/codeql/codeql-config.yml`

### Dependent Files

- `packages/site/lib/__tests__/public-install-contract.test.ts` — canonical shipped release/install contract; extend it for release header/footer and cliff identity rather than adding a duplicate static suite
- `MIGRATION_GUIDE.md` and the Task 09 site guide — install/upgrade guidance must match the beta channel truth documented here
- `docs/qa/` — post-publish beta checks are recorded as new `untested` scenarios, distinct from Task 13's pre-publish evidence

### Related ADRs

- [ADR-005: Distribution Identity — MIT Metadata and Active Legacy Channels](adrs/adr-005.md) — channel identities, the metadata-correction framing, pipeline retention, the single-contract installer, and the Go library close-out

## Deliverables

- Release configuration producing `compozy` artifacts under the legacy channel identities
- Dispatchable beta, stable, and legacy-maintenance release paths pinned to `releasepr v0.0.24`, consuming one authoritative plan while beta leaves npm `latest` and Homebrew untouched
- Repinned release slug and cosign identity with installer contract checks green, a single-contract `install.sh` targeting the documented beta, plus a clean-environment proof of the canonical Go install target
- Front-door README with the beta install section, `legacy/v0.2` deprecation pointer, preserved anchors/badges, and an OS-first body
- New front-door brand imagery and the authored single-cut + legacy-maintenance runbooks with their post-publish live-check lists
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from [`_tests.md`](_tests.md), the test contract — read each ID's full definition there before writing tests.

- [x] IT-017 — release identity: fresh release-PR dry-run evidence proves `goreleaser check`; workflow and vendored-skill pins resolve to `github.com/compozy/releasepr@v0.0.24`; `pr-release plan` accepts the explicit candidate ref, `version=0.3.0-beta.1`, and `channel=beta`; resolves the ref to checked-out `HEAD`; rejects `version=v0.3.0-beta.1`; rejects a derived tag present locally or on `origin`; and emits the complete authoritative output set. The workflow consumes those outputs without re-derivation, passes unprefixed `0.3.0-beta.1` to GoReleaser/npm, retains annotated tag creation, applies GitHub prerelease/npm `beta`/Homebrew-skip policy, exposes no AUR publisher, and passes installer contract fields (release repository slug, cosign identity regexp, renamed environment variables). Companion `stable` and `legacy` cases prove their latest/npm `latest`/Homebrew-publish policy without moving legacy artifact identity into the planner. It runs in the release-PR dry-run lane per repo CI policy, never on a schedule; external publish, registry, installer, and cosign acceptance remain post-publish runbook checks.

### Web/Docs Impact

- `web/`: none — checked surfaces: `web/src/**`, build output paths; reason: release packaging consumes the built SPA without changing it. The README image is a capture of the existing UI, not a UI change.
- `packages/site`: `public/install.sh` (hosted by the site, single v0.3 contract) and the front-door README that `public-install-contract.test.ts` reads. Site install/version copy stays consistent with the README's beta channel truth; landing/launch content is Task 11.
- QA impact: new scenarios — add content-addressed `untested` files for the beta install paths (npm `@beta`, hosted installer, versioned `go install`) and cosign verification against the new identity; reset scenarios whose `entry_points` cite install commands or self-update instructions.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked surfaces: extension manifests, hooks, skills/tools/resources, bundles, registries, bridge SDKs; reason: this task changes packaging and distribution identity, not runtime extension points.
- Agent manageability: `compozy update`/self-update paths report the new channel identities with unchanged structured output; the runbook is human-executed and explicitly out of the agent-operated surface. No verbs, endpoints, or error contracts change.
- Config lifecycle: no `config.toml` keys change — checked surfaces: `[update]` settings, installer environment variables; reason: installer variables are process environment (renamed with task 02's namespace), not configuration keys.

### AGH Impact Audit

- Native tools: no impact — checked tool IDs, toolsets, descriptors, schema digests, capability gates, and CLI/API fallbacks; distribution planning does not alter runtime tool contracts.
- Extensibility and hooks: no impact — checked extensions, hooks, skills/capabilities, tools/resources, bundles, registries, bridge SDKs, MCP sidecars, and config lifecycle; packaging identity does not change extension wiring.
- Workspace data isolation: no impact — checked workspace/session/agent scope, CLI/HTTP/UDS/core/store/web/SSE/cache/event propagation; no runtime datum or route changes.
- Official AGH skill: update the renamed `skills/compozy/` installation/update guidance for the root Go module, npm `beta` dist-tag, and hosted installer — Homebrew omitted until 0.3.0 stable — plus the `legacy/v0.2` boundary; no tool IDs, hook events, capabilities, bundles/resources, or memory/network/task semantics change.

## Success Criteria

- Every assigned test case implemented and passing
- `make installer-check`, `make codegen-check`, `goreleaser check`, and `make verify` green against the new slug and identity
- Repository and published manifests are MIT; any stale `BSL-1.1` survivor is corrected, while already-correct metadata is verified without fake edits
- Fresh IT-017 release-PR evidence proves the pinned `releasepr v0.0.24` ref/HEAD and local/remote tag guards, unambiguous version/tag normalization, authoritative beta prerelease/npm `beta`/Homebrew-skip outputs, and workflow-owned annotated tag creation without claiming an external publish
- `install.sh` implements exactly one verification contract and one documented beta target; no dual-path, fallback, or legacy PEM handling exists anywhere in it
- Every shipped install surface owned or touched here points at the beta channel and none offers Homebrew; the tap deprecation step is in the runbook; a beta build's self-update never offers the v0.2.x stable
- The single-cut runbook covers the full ordered sequence (legacy branch, squash merge, publish, domain pointing, retirement, archival, collateral, deprecation, live checks) with every irreversible step marked human-executed and gated on Task 13
- Every legacy README anchor has an explicit preserved/mapped/no-successor disposition, the beta install section and `legacy/v0.2` pointer are complete, all semantic successor anchors and the badge row work, release templates/cliff identity are current, and `public-install-contract.test.ts` passes
