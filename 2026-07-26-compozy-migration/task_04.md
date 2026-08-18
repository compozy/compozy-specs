---
status: completed
title: Rebrand workspace packages, web, documentation site, and official runtime skill
type: frontend
complexity: high
---

# Task 04: Rebrand workspace packages, web, documentation site, and official runtime skill

## Overview

Fourth rebrand class: the npm workspace scope (`@agh/*` → `@compozy/*`), the web SPA branding, documentation-site brand/SEO layer, public OpenAPI artifact names, official runtime skill rename (`skills/agh/` → `skills/compozy/`), and elimination of the "Artificial General Hivemind" expansion. Generated content regenerates from its owners rather than being hand-swept; only pinned specs whose asserted contract changes are edited, while the complete suite runs.

<critical>
- ALWAYS READ `_brief.md`, `_techspec.md`, `_content-plan.md`, `_tests.md`, every ADR, and any per-task memory before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — package, OpenAPI, skill metadata, and blog-slug renames are hard cuts with no aliases, duplicate exports, old loader key, or redirect-by-fallback
</critical>

<requirements>
- MUST rename every affected npm workspace package from `@agh/*` to `@compozy/*` and update imports, package templates, examples, and the Bun lockfile across `web`, `packages/{ui,site}`, `sdk/typescript`, `sdk/create-extension`, and `sdk/examples/prompt-enhancer`. Rename `AGH_CODE_THEMES` and its type aliases with the shared UI package. The destination SDK/scaffolder are source-distributed workspace packages only in v0.3.0: mark `sdk/typescript` and `sdk/create-extension` private and do not publish them over the already-public, API-incompatible legacy `@compozy/extension-sdk@0.1.x` and `@compozy/create-extension@0.1.x` identities.
- MUST update `packages/site/lib/site-config.ts` brand name and description and collapse the `%s | AGH` title template, setting `https://compozy.com` as the effective URL, metadataBase, canonical, sitemap, robots, RSS, llms, and OpenGraph origin — the site ships as `compozy.com` from the single cut (brief round-11); no staged cutover exists. Task 11 verifies the full SEO cascade with the launch content.
- MUST hard-cut `openapi/agh.json` → `openapi/compozy.json` and `web/src/generated/agh-openapi.d.ts` → `compozy-openapi.d.ts`, then update their generator sources, imports, tests, and site OpenAPI reader before regeneration. Preserve the separate `compozy-daemon` artifact only where its existing contract proves it is distinct.
- MUST regenerate — never hand-sweep — the CLI reference tree, API reference tree, and changelog receipts from their sources. Use globs and generator ownership rather than stale page counts; preserve authored `api-reference/index.mdx` as authored content.
- MUST eliminate the "Artificial General Hivemind" expansion from the CLI root command, the landing hero eyebrow, OG templates, goreleaser descriptions, and generated docs.
- MUST rename `skills/agh/` → `skills/compozy/`, set `name: compozy`, rename the loader-read `metadata.agh.*` frontmatter namespace, rename `references/contributing-to-agh.md`, and update package/OpenAPI/metadata references owned by this task. Verify that the command/env/protocol body hard cuts from tasks 01–03 remain clean; do not defer or duplicate their ownership.
- MUST rename the runtime skill loader namespace from `metadata.agh.*` to `metadata.compozy.*` in loader types, declarations, normalization, diagnostics, fixtures, and canonical loader/CLI/bundled-skill suites. No legacy metadata reader is allowed.
- MUST deliver an interim plain-type "Compozy" lockup in `packages/ui/src/components/custom/logo.tsx` and its duplicate in `packages/site/lib/og/logo.tsx` — both are SVG path geometry with no text literal, so this is a redraw, not a string swap. The drawn wordmark is task 11.
- MUST route visible web/site/shared-UI work through the `designer` agent in execution mode, activate `eng-design`, `ui-craft`, and `impeccable` before editing, and verify every changed UI surface with `eng-ui-screenshot` evidence.
- MUST update every affected pinned spec in `packages/site/lib/__tests__/` in the same commit as the surface it pins, and run the complete suite discovered from the repository at execution time. Do not mutate unaffected specs merely because they share a directory.
- MUST hand-author the hero rewrite ONLY as far as removing the retired expansion; the locked headline and definitional block land in task 11 with the launch content.
- MUST regenerate `DESIGN.md` through its generator inputs (`scripts/sync-design-md.mjs` prose, frontmatter `name`, `tokens.css`) and MUST NOT hand-edit generated regions (L-024); the `.agh-mermaid` CSS class is code-coupled and renames mechanically.
- MUST NOT touch `web/` route behavior, payloads, or controls — branding only.
- MUST update developer docs and the migration guide handoff to say the legacy public SDK/scaffolder stay frozen on their v0.1.x line with no v0.3 registry successor; current Compozy extension development uses the version-matched repository checkout. Do not claim an API-preserving package rename.
- MUST add the specific launch-post slug redirect in `packages/site/next.config.mjs` when its blog slug changes, updating the blog map, cover asset reference, changelog link, and affected specs. This same-domain slug mapping is permanent site configuration and the only redirect entry — no cross-domain redirect infrastructure exists (brief round-11).
</requirements>

## Subtasks

- [x] 4.1 Rename every affected workspace package and import, including the SDK workspaces, package templates, Bun lockfile, and shared code-themes constant; make the colliding SDK/scaffolder packages explicitly private
- [x] 4.2 Update site config brand identity and collapse the title template; set `compozy.com` as the canonical origin and verify the SEO cascade
- [x] 4.3 Hard-cut the OpenAPI artifact names and all source readers, then regenerate their TypeScript output from the renamed specs
- [x] 4.4 Regenerate CLI and API reference trees from corrected generator inputs; update root CLI/docpost metadata and confirm diff-clean
- [x] 4.5 Sweep authored site docs (`runtime/core`, `guides`, `use-cases`) for brand/env/path tokens
- [x] 4.6 Sweep web SPA branding (title, meta, chrome strings) with no behavior change
- [x] 4.7 Eliminate the retired expansion across CLI, hero eyebrow, OG templates, and generated output
- [x] 4.8 Rename the official runtime skill directory, loader namespace, frontmatter namespace, and reference bodies with no legacy metadata reader
- [x] 4.9 Redraw the interim plain-type lockup in both logo sources
- [x] 4.10 Regenerate `DESIGN.md` via generator inputs and rename the code-coupled CSS class
- [x] 4.11 Update only affected pinned site specs, run the complete site-spec suite, and add the launch-post slug redirect with its linked assets and receipts
- [x] 4.12 Run `make codegen-check`, the Bun lanes, and the class-scoped grep gates

## Implementation Details

Classification drives the work: MECHANICAL token sweeps, REGEN for generator-owned trees, and a small REWRITE surface limited to removing the retired expansion. Follow `_content-plan.md` §B (B1-B9) and TechSpec §Development Sequencing step 1. The OpenAPI filenames are a public-codegen hard cut: rename the source specs and generated declaration, update every reader/import/template first, then regenerate. Do not retain an `agh` source filename, a compatibility import, or a fallback site reader.

The blog launch post's slug rename needs its own redirect entry in `packages/site/next.config.mjs`; update the blog map, cover-asset reference, changelog receipt, and the affected specifications with it. There is no separate domain redirect (brief round-11) — this slug mapping is the only redirect entry.

The skill metadata rename is a loader contract change, not only frontmatter text. Update `internal/skills/metadata.go` and associated normalization, diagnostics, fixtures, declaration, loader, CLI, and bundled-skill suites to accept only `metadata.compozy.*`.

### Relevant Files

- `packages/site/lib/site-config.ts:2-7` — `name: "AGH"` and description embed the retired identity; the URL becomes `https://compozy.com`, the canonical origin from the single cut
- `packages/site/app/layout.tsx` — `%s | AGH` template, metadataBase, OG alt
- `packages/site/components/landing/hero.tsx:48-58` — accent brand span, the retired expansion, hero copy (headline relock is task 11)
- `packages/site/lib/og/templates/{landing,docs,blog}.tsx` + `lib/og/logo.tsx` — eyebrows, domain strings, SVG lettering geometry
- `packages/site/lib/__tests__/**` — complete pinned-spec suite; update only the specifications whose asserted surface changes, including metadata, install, blog, OpenAPI, and skill contracts
- `skills/agh/**` — official runtime skill source being hard-cut to `skills/compozy/**`
- `internal/skills/{metadata.go,loader*.go,metadata*_test.go}` and bundled-skill/CLI suites — loader-read `metadata.agh.*` namespace, diagnostics, fixtures, and discovery contract
- `packages/ui/src/components/custom/logo.tsx`, `packages/ui/src/index.ts` — primitive inventory and lockup source of truth
- `openapi/agh.json`, `web/src/generated/agh-openapi.d.ts`, `magefiles/defaults.go`, `internal/api/spec/spec.go` — OpenAPI source/output names and generator/read paths
- `packages/site/lib/**`, `web/src/**`, `sdk/**` — OpenAPI readers/imports, package templates, and workspace package references
- `internal/cli/root.go`, `internal/cli/docpost/**` — root command and generated-reference metadata
- `web/index.html`, `web/src/**` — title/meta and brand strings

### Dependent Files

- root `package.json`, `bun.lock`, `web/package.json`, `packages/{ui,site}/package.json`, `sdk/{typescript,create-extension,examples/prompt-enhancer}/package.json` — workspace identity, package templates, and explicit non-public status for the API-incompatible SDK/scaffolder collisions
- `sdk/{extension-sdk-ts,create-extension}/package.json` — existing public `@compozy/*@0.1.x` identities that v0.3.0 must not overwrite
- `packages/site/source.config.ts`, `packages/site/velite.config.ts` — documentation-site identity
- `packages/site/content/blog/posts/introducing-agh-*.mdx` — launch post slug rename + redirect entry
- `DESIGN.md` — regenerated output; generator inputs are the edit surface
- `web/CLAUDE.md`, `packages/ui/CLAUDE.md`, `packages/site/CLAUDE.md` — skill/package names; full prose sweep is task 05, but lines naming renamed packages co-ship here

### Related ADRs

- [ADR-004: Dev-Cycle Bundles Nine Skills From `.agents/skills`; Authoring Runs in Managed Sessions](adrs/adr-004.md) — the official skill rename to `skills/compozy/` and the namespace collision that retires the legacy `compozy` skill

## Deliverables

- `@compozy/*` workspace packages, including private source-distributed SDK templates/examples, with every import resolved and the Bun lockfile regenerated without publishing over legacy registry identities
- Site identity, SEO cascade, and OG surfaces carrying the new brand with the retired expansion gone
- `openapi/compozy.json` and `web/src/generated/compozy-openapi.d.ts` with all codegen/read paths hard-cut and regenerated
- Regenerated CLI/API reference trees and changelog receipts, diff-clean under `make codegen-check`
- `skills/compozy/` plus a `metadata.compozy.*` loader contract, fixtures, and diagnostics with no legacy reader
- Interim plain-type lockup in both logo sources
- All pinned site specs passing; only affected specifications changed in the same commit as their surface
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from [`_tests.md`](_tests.md), the test contract — read each ID's full definition there before writing tests.

- [x] No new `_tests.md` IDs are assigned to this task. Coverage is owned by the complete existing pinned site-spec suite (only affected specs change), canonical skill loader/CLI/bundled-skill suites, `make codegen-check` for regenerated trees, the Bun lint/typecheck/test lanes, and class-scoped grep gates. Per repo test-placement policy, no standalone prose/brand-string suites are added — existing contract suites already own these invariants.

### Web/Docs Impact

- `web/`: `web/index.html` (title/meta), brand strings across `web/src/**`, `@agh/ui` → `@compozy/ui` imports in every consuming component, Storybook configuration, and renamed generated OpenAPI imports. No route, hook, query-key, payload, or control changes.
- `packages/site`: `lib/site-config.ts`, `app/layout.tsx`, `components/landing/**`, `lib/og/**`, `lib/footer-config.ts`, `content/runtime/{core,guides,use-cases}/**`, regenerated `content/runtime/{cli-reference,api-reference}/**`, `content/blog/**`, `public/site.webmanifest`, llms route headers, OpenAPI readers, and the affected specs under `lib/__tests__/`.
- QA impact: reset to `untested` every scenario whose `entry_points` cite site URLs, the web title/chrome, or the official skill name; add a content-addressed `untested` file for the renamed skill. The private SDK workspace names are not presented as new registry channels; public CLI npm identity belongs to task 10.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: the official bundled runtime skill is renamed (directory, `name`, loader-read `metadata.compozy.*` namespace, and reference bodies) — this is an extension-surface rename that the skill loader reads at boot. The AGH TypeScript SDK/scaffolder have a different API from the legacy public packages with the same future names, so they remain private, version-matched workspace artifacts in v0.3.0; no npm successor or compatibility claim is created. No hook or bundle semantics change.
- Agent manageability: `compozy skill list/view` must resolve the renamed official skill; generated CLI reference regenerates from the cobra tree so agent-facing help stays truthful. No new verbs, endpoints, or structured-output shapes.
- Config lifecycle: no `config.toml` key changes — checked surfaces: skill configuration keys, `[web]` settings; reason: the rename touches package and content identity, not configuration.

### AGH Impact Audit

- Native tools: native tool IDs, toolsets, descriptors, schema digests, and capability gates are unchanged; checked the OpenAPI/codegen and CLI-reference surfaces because this task only renames their artifacts and brand metadata.
- Extensibility and hooks: the bundled official skill directory and loader-read metadata namespace hard-cut to `compozy`; checked extension manifests, hooks, bundles, registries, bridge SDKs, and package templates. No hook event or bundle semantics change.
- Workspace data isolation: no impact — checked workspace/session/agent data models, CLI/HTTP/UDS propagation, SSE, cache, and events; this task changes names, generated artifacts, and documentation identity only.
- Official AGH skill: rename `skills/agh/**` to `skills/compozy/**`, hard-cut its name and `metadata.compozy.*` loader contract, update Task-04-owned package/OpenAPI references, and verify the live command/env/protocol guidance already co-shipped by tasks 01–03 remains clean.

## Success Criteria

- Every assigned test case implemented and passing
- `make codegen-check` diff-clean, `bunx turbo run lint typecheck test` green for `./web`, `./packages/ui`, and `./packages/site`, and final `make verify` green
- Zero old package/OpenAPI/skill identifiers in executable, workspace-manifest, generator, loader, and changed fixture scopes; deferred authored docs are verified by task 05's separate body gate
- The SDK/scaffolder workspace manifests are private, no release workflow publishes them, and migration guidance explicitly distinguishes them from the frozen legacy public packages
- The site builds, the complete pinned-spec suite passes, and each changed specification asserts a changed surface
