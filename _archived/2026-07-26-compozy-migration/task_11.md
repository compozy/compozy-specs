---
status: completed
title: Launch content in-branch — landing, wordmark, launch post, release notes
type: docs
complexity: medium
---

# Task 11: Launch content in-branch — landing, wordmark, launch post, release notes

## Overview

Implements the launch content directly on the `v0.3` branch as part of the single cut: the locked hero and launch copy, the drawn Compozy wordmark replacing the interim lockup, the six-slot landing composition with an explicit disposition for every current section, the static OS-shell hero capture replacing the Remotion player, the launch blog post, and "The OS Release" notes. Everything ships in the tree that Task 13 validates and the single-cut runbook publishes — there is no isolated change set, no deferred activation, no banner, and no separate stable-promotion stage (brief round-11). The site describes v0.3 as beta wherever install/version status appears, as ordinary copy updated by the post-beta stable release outside this spec.

<critical>
- ALWAYS READ `_brief.md` (rounds 6 and 11 govern content and staging), `_techspec.md`, `_tests.md`, `_content-plan.md`, `_tasks.md`, and the applicable ADRs before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — do not use a temporary landing claim, an interim logo, an unverified competitor assertion, or a deferred/hidden variant of any launch surface
</critical>

<requirements>
- MUST be the sole owner of the hero, wordmark, and launch copy. Implement the hero exactly as locked in `_brief.md` round-6 — headline "The only true OS for AI agents." plus the definitional test block, verbatim. This is locked text to implement, not copy to author. It ships live on the branch; no activation gate exists.
- MUST structure the landing page as: hero → "Built to be built on" (extensibility as the second criterion of "true") → feature wall → market comparison → proof → CTA, per the round-6 material allocation.
- MUST reconcile the current twelve-section landing explicitly. Retain/reshape `Hero`, `ExtensibilitySection`, `Comparison`, and `FinalCta`; merge truthful material from `SupportedAgents`, `NetworkSection`, `BentoSection`, `MemoryDreamSection`, `AutonomyKernelSection`, `FeaturesSection`, `BridgesSection`, and `InstallSection` into the feature-wall/proof slots; then delete superseded component files/exports rather than leaving an unreachable parallel landing implementation. Also disposition the already-unreachable `RuntimeSection`, `SandboxSection`, and `RuntimeMicroDiagram`: merge any unique truthful material into the approved slots, then remove their files, barrel exports, and isolated tests.
- MUST sell completeness as integration, never as a feature list, and MUST NOT let "true"/"only" appear without the adjacent definitional test or named subsystems.
- MUST use an OS-shell capture (windows, task board, loop run) as the hero image — a generic dashboard capture invalidates the claim.
- MUST replace the current Remotion hero player with that static verified OS-shell capture. Delete `components/landing/hero-player.tsx` and `packages/site/remotion/hero/**`, remove `@remotion/player`/`remotion` from `packages/site/package.json` and `bun.lock` when no other consumer exists, remove the retired Remotion entries from `knip.json`, and update the owning landing/heading specs, `packages/site/CLAUDE.md`, and the final-QA site seed so none describes or mocks the deleted player. Dead motion code, dependencies, instructions, and QA assertions are delete targets, not retained alternatives.
- MUST deliver the drawn Compozy wordmark in both logo sources (the shared UI primitive and its OG duplicate), replacing task 04's interim plain-type lockup.
- MUST keep the site's install/version copy truthful to the beta channel: every install surface documents `@compozy/cli@beta`, the hosted installer's beta target, and versioned `go install`, consistent with Task 10's front-door README, and no install surface offers Homebrew until the post-beta 0.3.0 stable bump (brief round-11); `compozy.com` is the canonical origin across metadataBase/canonical/sitemap/robots/RSS/llms/OG (verify the executed task 04 state — no `agh.network` origin survives). Beta status is ordinary copy; no banner or removable artifact is introduced.
- MUST relock `COPY.md` hero/subhead and `packages/site/CLAUDE.md` to the shipped text if any wording drifted during implementation.
- MUST author the launch blog post and the "The OS Release" notes as editorial narrative over the generated changelog, with honest breaking-change framing, the beta channel truth (what installing the beta means, where legacy lives), and the license metadata-correction note (never a relicense).
- MUST make named competitor claims only with an exact source path recorded below; otherwise remove the named comparison and retain factual Compozy subsystem inventory. Do not claim that a competitor has no equivalent without source-supported scope.
- MUST verify Task 04's renamed launch-post slug mapping remains valid — it is same-domain permanent site configuration and the only redirect entry that exists; no cross-domain redirect infrastructure is created (brief round-11).
- MUST keep every claim within the repo's claim standards — exact numbers only, nothing unshipped named.
- MUST update all affected pinned site specs in the same commit as their surfaces.
- MUST route all landing/logo/launch visual work through the `designer` agent in execution mode, activate `eng-design`, `ui-craft`, and `impeccable` before editing, and verify every changed UI surface with `eng-ui-screenshot` evidence.
- MUST NOT create activation gates, deferred or isolated change sets, banner artifacts, redirect infrastructure, or a stable-promotion runbook — the post-beta stable release (npm `latest`, Homebrew, copy re-tag) is normal release work through Task 10's workflow, outside this spec.
</requirements>

## Subtasks

- [x] 11.1 Implement the locked hero headline and definitional block live on the landing page
- [x] 11.2 Build the "Built to be built on" extensibility section and record the retain/merge/delete disposition for all current landing sections
- [x] 11.3 Build the feature wall and only source-backed market comparison content; remove superseded landing components/exports
- [x] 11.4 Capture the OS-shell hero image, replace/delete the Remotion player stack, and wire the proof/CTA sections
- [x] 11.5 Draw and ship the Compozy wordmark in both logo sources
- [x] 11.6 Relock `COPY.md` hero/subhead and the site hero lock to the shipped text
- [x] 11.7 Author the launch blog post and "The OS Release" notes with beta channel truth and the license metadata-correction note
- [x] 11.8 Align site install/version copy with the beta channel and verify the `compozy.com` canonical origin and Task 04's renamed-slug mapping
- [x] 11.9 Update affected pinned site specs, run the Bun lanes, and finish with `make verify`
- [x] 11.10 Verify the organization profile repository; update it when authorized, otherwise record `not applicable — absent or unavailable` without blocking the cut

## Implementation Details

Follow `_brief.md` round-6 (locked hero, thesis, material allocation, craft rules), round-11 (single-cut staging: launch surfaces ship in-branch), `_content-plan.md` B12/C13, and TechSpec §Development Sequencing steps 5-6. Activate the design and copy skills before touching any visible surface, and verify the rendered result with a screenshot capture.

The market comparison may draw on the competitive reading only when each rendered assertion is traceable to the source paths below. If that traceability is absent, omit the competitor-specific assertion; the Compozy subsystem inventory remains factual and must still meet the claim standard.

### Evidence Sources for Market Claims

- `.resources/orca/src/main/ghostty/index.ts` — Orca terminal/window integration evidence
- `.resources/paperclip/README.md` — Paperclip product-scope evidence
- `.resources/smithers/README.md` — Smithers product-scope evidence
- `.resources/openclaw/README.md` — OpenClaw product-scope evidence
- `.resources/synara/apps/marketing/src/pages/index.astro` — Synara public positioning evidence
- `.resources/t3code/README.md` — T3 Code product-scope evidence

Read and cite only sources that exist at implementation time. Remove any row whose exact source is absent or does not support the specific claim.

### Relevant Files

- `packages/site/components/landing/**` — hero and landing sections carrying the locked text, including the unreachable `runtime-section.tsx`, `sandbox-section.tsx`, `runtime-micro-diagram.tsx`, and their barrel/test references that require an explicit disposition
- `packages/site/app/(home)/page.tsx` — current twelve-section composition and the sole landing-order owner
- `packages/site/components/landing/hero-player.tsx`, `packages/site/remotion/hero/**`, `packages/site/package.json`, `bun.lock`, and `knip.json` — Remotion player code, dependencies, and analysis entries to remove after the static OS-shell capture replaces them
- `packages/ui/src/components/custom/logo.tsx` and `packages/site/lib/og/logo.tsx` — the two wordmark sources (SVG geometry, no text literal)
- `packages/site/lib/og/templates/{landing,docs,blog}.tsx` — OG copy aligned to the launch positioning
- `packages/site/lib/site-config.ts` — canonical `compozy.com` origin (flipped by executed task 04; verify the full SEO cascade)
- `COPY.md` — hero/subhead locks; `packages/site/CLAUDE.md` — the site-side hero lock and retired Remotion stack description
- `packages/site/content/blog/posts/**` — launch post content; verify, but do not re-own, Task 04's renamed-slug redirect entry
- `.release-notes/**` and the GitHub release body — "The OS Release" narrative over the generated changelog
- `packages/site/components/landing/__tests__/landing.test.tsx`, `packages/site/lib/__tests__/public-heading-hierarchy.test.tsx`, and other affected `packages/site/lib/__tests__/**` owners — remove obsolete player mocks and update only contracts whose rendered surface changes

### Dependent Files

- `docs/design/` captures — hero image source material
- `docs/qa/_seeds/final-qa/_children/13-docs-site.md` — replace Remotion-specific diagnostics/code references with the static OS-shell hero contract
- `MIGRATION_GUIDE.md` and the site migration section — linked from the launch post and release notes
- `README.md` (root) — Task 10's front door; body and install copy align with the launch positioning and beta channel truth

### Related ADRs

- [ADR-005: Distribution Identity — MIT Metadata and Active Legacy Channels](adrs/adr-005.md) — the metadata-correction framing and channel truth that launch copy must use

## Deliverables

- Landing page implementing the locked six-slot composition live on the branch, with every prior section retained/merged/deleted explicitly and no unreachable duplicate implementation
- Drawn Compozy wordmark in both logo sources with the interim lockup removed
- Launch blog post, "The OS Release" notes, and relocked copy authorities carrying beta channel truth
- Site install/version copy consistent with the front-door README, `compozy.com` canonical origin verified, and Task 04's same-domain slug mapping intact as the only redirect entry
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from [`_tests.md`](_tests.md), the test contract — read each ID's full definition there before writing tests.

- [x] No new `_tests.md` IDs are assigned to this task. Coverage is owned by the existing pinned site specs updated in this change (landing truth, OG image, blog metadata, navigation, search index, install contract), the Bun lint/typecheck/test lanes, and a screenshot capture of the rendered landing page cited in the completion notes. Prose and CSS tests remain forbidden by repo policy.

### Web/Docs Impact

- `web/`: none — checked surfaces: `web/src/**`; reason: launch content lives in the marketing/docs site and shared UI logo primitive. The hero image is a capture of the shipped OS shell, not a UI change. The wordmark change in `packages/ui` propagates to any web surface rendering the logo — verify visually.
- `packages/site`: `components/landing/**`, `lib/og/**`, `content/blog/**`, install/version copy, and `site-config` origin verification; the affected specs under `lib/__tests__/` remain the canonical site coverage.
- QA impact: add content-addressed `untested` scenarios for the landing hero, the beta install copy, and blog/launch-post routes; reset scenarios whose `entry_points` cite the previous hero copy or the retired `agh.network` origin.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — checked surfaces: extension manifests, hooks, skills/capabilities, tools/resources, bundles, registries; reason: launch content is marketing and documentation. The "Built to be built on" section *describes* the extensibility surfaces shipped in tasks 06-08 without changing them; every claim must trace to a shipped mechanism.
- Agent manageability: none — checked surfaces: CLI verbs, HTTP/UDS routes, structured output; reason: no runtime surface changes.
- Config lifecycle: none — checked surfaces: `config.toml` keys and defaults; reason: content-only change. Hosting and DNS are runbook execution (Task 10), not runtime config.

### AGH Impact Audit

- Native tools: no impact — checked tool IDs, toolsets, descriptors, schema digests, capability gates, and CLI/API fallbacks; marketing surfaces do not change runtime tool contracts.
- Extensibility and hooks: no impact — checked extensions, hooks, skills/capabilities, tools/resources, bundles, registries, bridge SDKs, MCP sidecars, and config lifecycle; this task describes but does not rewire those surfaces.
- Workspace data isolation: no impact — checked workspace/session/agent scope, CLI/HTTP/UDS/core/store/web/SSE/cache/event propagation; no runtime datum or route changes.
- Official AGH skill: no impact — checked the renamed `skills/compozy/`; launch copy and logo geometry do not change public tool IDs, CLI paths, hook events, capabilities, bundles/resources, or memory/network/task semantics.

## Success Criteria

- Every assigned test case implemented and passing
- The rendered landing page carries the locked hero verbatim, with an OS-shell hero image, verified by a cited screenshot capture
- The drawn wordmark ships in both sources with no interim lockup remaining
- Site install/version copy matches the beta channel truth and the front-door README (beta paths only; no Homebrew offered); the `compozy.com` canonical origin holds across the SEO cascade; Task 04's renamed launch-post slug mapping remains intact without duplicate ownership
- Every affected pinned site spec passes in the same commit; Bun lanes and `make verify` are green
- No superseded landing component, `hero-player`, Remotion hero source, unused Remotion dependency, obsolete test mock, activation gate, banner, or isolated change set remains
