# Remove the unused bundle product surface and make Extension the single kit unit (through P1 robustness)

## Problem

Compozy exposes two user-facing concepts for shipping add-ons: **Extension** (install/enable) and **Bundle** (catalog/activate/deactivate). That split creates a dual mental model plus CLI, HTTP/UDS, native tools, marketplace UI, docs, QA, and skills surface — but first-party product content never uses bundles.
The only shipped kits today are extensions (`dev-cycle`, `bridges/*`). They already turn on via enable. Bundles were framed as “outcome presets,” yet the implemented bundle is a narrower **activation projector** owned by one already-installed extension: it does not install dependencies, bind secrets, or enable bridge delivery. No first-party extension ships a bundle profile. Docs and marketplace examples name bundles that do not exist in-repo.
Product decision: **hard-delete the public Bundle concept** and make **Extension** robust enough to be the only kit unit, covering the useful outcomes that bundles claimed (through an agreed P0+P1 capability set), without reintroducing pack/compose ceremony.
Outcome wanted: operators and agents manage one concept — `extension install → (bind secrets/env if needed) → enable → kit live; disable → kit off` — with no `compozy bundle *`, no `/api/bundles/*`, no marketplace kind `bundle`, and no `compozy__bundles_*` tools.

## Current State

### Extension (live kit path)

- Manifest static resources: `skills`, `loops`, `agents`, `bundles`, `hooks`, `tools`, `mcp_servers`, plus publish grants (`internal/extension/manifest.go` `ResourcesConfig`).
- Enable loads static skills/loops/agents/hooks/tools/MCP. Reference kit: `extensions/dev-cycle/` (skills, loops, static agents, `tool.provider` tools) — **no bundles**.
- Bridge kits: `extensions/bridges/{slack,github,...}/` provide `bridge.adapter` only; instances/secrets/enable are separate operator steps.
- Install/enable/disable/update/remove, trust/provenance, and marketplace install are mature **per-extension** (CLI, HTTP/UDS, `compozy__extensions_*`). Batch exists for update (`--all`), not for install/compose of N extensions.

### Bundle (second ceremony)

- Schema: `BundleSpec` / `BundleProfile` with channels, agents (+ soul/heartbeat sidecars), layouts, jobs, triggers, bridge presets (`internal/extension/bundle.go`).
- Runtime package: `internal/bundles/**` — activate/preview/materialize; resource kinds `bundle` and `bundle.activation` in `resource_records` (legacy `bundle_activations` tables removed).
- Bridge materialization forces disabled instances (`internal/bundles/service_materialize_bridge.go` sets `Enabled: false`, `Status: Disabled`).
- Explicit non-capabilities (docs): does not install other extensions, resolve deps, bind secrets, health-check deps, or enable bridge delivery; skills/loops/hooks/tools/MCP stay extension-scoped (`docs/ecosystem/bundle-opportunities.md`).
- Public surfaces: CLI `compozy bundle *`, HTTP/UDS `/api/bundles/*` (catalog, preview, activations, network/settings), native tools `compozy__bundles_{list,info,activate,deactivate,status}`, web `/marketplace/bundles` and `/marketplace/bundles/activations/$id`, marketplace kind `bundle`, docs under `resources/bundles.mdx` / `cli/bundle/*` / `api/bundles.mdx`.
- Extension disable/remove blocked while activations exist (`ErrExtensionHasActiveBundles`).
- Repo truth: “No first-party extension in this repository currently ships a bundle profile” (`docs/ecosystem/bundle-opportunities.md:9`). No committed first-party `bundles/*.toml`; tests write TOML inline or in e2e temp dirs.

### Asymmetry that motivates the cut

| Surface                                         | On extension enable         | Via bundle activate         |
| ----------------------------------------------- | --------------------------- | --------------------------- |
| Skills, loops, static agents, hooks, tools, MCP | Yes                         | No selective control        |
| Soul/Heartbeat packaging on agents              | Incomplete on static agents | Yes on profile agents       |
| Jobs / triggers (file-declared)                 | No static dir loader        | Yes (agent-targeted only)   |
| Window layouts (file-declared)                  | No static dir loader        | Yes                         |
| Bridge instances                                | Adapter only                | Disabled presets only       |
| Network declared channels + confirm UX          | Manifest digest exists      | Confirm wired to activation |

## Evidence

- `docs/ecosystem/bundle-opportunities.md:9` — zero first-party bundle profiles shipped.
- `docs/ecosystem/bundle-opportunities.md:20-40` — what a profile can project vs platform-evolution gaps.
- `internal/extension/bundle.go` — profile schema (agents/layouts/jobs/triggers/bridges/channels).
- `internal/extension/manifest.go:54-63` — static resources include `bundles` but not jobs/layouts dirs.
- `internal/bundles/service_materialize_bridge.go:68-69` — projected bridges always disabled.
- `extensions/dev-cycle/extension.json` — real kit ships without bundles.
- `internal/tools/builtin_ids.go:362-370` — `compozy__bundles_*` tool IDs; toolset `compozy__bundles`.
- OpenAPI ops: `listBundleCatalog`, `previewBundleActivation`, `listBundleActivations`, `activateBundle`, `getBundleActivation`, `updateBundleActivation`, `deleteBundleActivation`, `getBundleNetworkSettings` under `/api/bundles/*`.
- QA scenarios `ET-024`–`ET-030`, `ET-web-bundle-*`, `NB-023` encode the product surface as living contract.
- Web e2e `web/e2e/__tests__/marketplace.spec.ts` exercises temp `bundles/*.toml` + activate/deactivate UI.

## Requirements

When this work is done, the following must be true:

### Hard cut — Bundle product removed

1. No public Bundle concept remains: no marketplace kind `bundle`, no bundle catalog/activation resources as product API, no `compozy bundle` CLI, no `/api/bundles/*` HTTP/UDS routes, no `compozy__bundles_*` native tools/toolset, no web marketplace bundle activate/detail routes or activation dialogs.
2. No extension authoring path that requires `resources.bundles` / bundle profiles / `bundle.activation` for a kit to work.
3. Docs, official skill (`skills/compozy`), README product language, OpenAPI, and CLI references stop teaching Bundle as a product unit.
4. Homonyms that are **not** this product must keep working: support diagnostics bundles (`/api/support/bundles*`, `compozy support bundle`), skill `SourceBundled` / `--source bundled` precedence, task ContextBundle, crash bundle paths, CLI `outputBundle` format helpers, sigstore/JS “bundle” wording.
5. Extension disable/remove no longer depends on “active bundle activations”; any ownership/cleanup semantics that still exist live under the extension lifecycle.
6. QA tracker impact: user-visible bundle scenarios are removed or rewritten against extension-only flows; no orphaned `compozy bundle` / `/api/bundles` entry points left as if the surface still ships.
7. Every surface in **Removal inventory** below is deleted, unwired, regenerated, or rewritten so greps for product Bundle APIs do not find living contracts (historical/archived notes may remain annotated).

### Extension robustness — P0 (kit completeness on enable)

1. An extension package can ship **complete agents** including authored-context sidecars (`SOUL.md` / `HEARTBEAT.md`) that become live when the extension is enabled, without a second activate step.
2. An extension package can declare **automation jobs and triggers** as package resources (same class of authoring as skills/loops), and enabling the extension makes those definitions available under clear ownership. Recurring work must **not** auto-start merely because the package was installed; enable (or an explicit consent step after enable) is the gate.
3. An extension package can declare **window layouts** as package resources that become available on enable under clear ownership.
4. **Enable is the kit switch:** for a well-formed kit extension, enable brings the shipped static kit surfaces online together; disable removes or unpublishes extension-owned kit resources without requiring a separate deactivate verb.
5. Install/enable surfaces report missing env/secret requirements and support a **post-install secrets/env binding path** comparable in agent/operator manageability to what MCP install already provides for vault bindings (extensions today largely surface `missing_env` names only).

### Extension robustness — P1 (operability without Bundle)

1. Operators and agents can inspect an **inventory of resources owned/projected by an installed extension** (what it ships vs what is currently live).
2. Operators and agents can **preview** the effect of enabling (or applying) an extension’s kit resources without mutating state (dry-run conflicts/ownership).
3. If an extension declares Live `network_participation` requirements, confirmation/digest handling is available on the **extension** lifecycle (not on a removed bundle activation object).
4. After `extension update`, package-owned kit resources reconcile with a defined drift/reapply story (no silent stale projections).
5. `capabilities.provides` is truthful: only provides that negotiate real Host API service methods are accepted/documented; decorative/cargo-cult provides are rejected or warned and docs/examples stop teaching them.

### Agent-manageability & co-ship

1. Anything removed from `compozy__bundles_*` that remains necessary for kit management is reachable via extension (and/or resources) CLI/HTTP/UDS/native-tool parity with structured output.
2. `packages/site` docs and generated API/CLI references co-ship with the hard cut (greenfield: no compat shims, aliases, or dual fields).
3. Official skill `skills/compozy` is updated for removed tools and an extension-only kit narrative.
4. Agent instruction checklists (`AGENTS.md`, `CLAUDE.md`, `internal/AGENTS.md`, cy-web-docs-impact / cy-spec-preflight triggers) no longer list product bundles as a living extensibility surface.

## Constraints & Non-goals

### Constraints

- Greenfield alpha: hard cut, delete obsolete code/docs/tests; no migration bridges, aliases, or “bundle means extension” compatibility layers.
- Do not grow god files; split by responsibility if touching large packages.
- Preserve bridge adapter install/setup as its own extension path; do not pretend enable auto-binds Slack/GitHub console secrets.
- Jobs/triggers shipped by extensions must remain consent-first (no surprise cron on install).
- Homonym “bundle” surfaces unrelated to extension kits must keep working.
- Cross-surface impact audit required at completion (native tools, extensibility, workspace isolation, official skill).
- Historical artifacts (`.codex/plans/*extension-bundle*`, QA bug reports, `_archived` tasks) may stay; annotate as pre–hard-cut, do not treat as living contract.
- Conversation artifacts (TechSpec/tasks if created later) in English; this brief is the handoff.

### Non-goals

- Do **not** build a Pack/compose product that installs N extensions, transactional multi-deps, or marketplace “outcome packs.”
- Do **not** reintroduce multi-profile activate/deactivate as a second public concept (even under a new name).
- Do **not** require selective per-profile skill/loop/tool scoping for this cut.
- Do **not** require workspace-scoped enable, Live channel enrollment, memory seeding, or task-graph starters as part of this brief’s P0/P1 bar (may be later).
- Do **not** expand third-party bridge SDK / marketplace bridge authoring beyond what is needed so deleting bundle bridge-preset projection does not break in-tree bridges.
- Do **not** keep an empty `internal/bundles` projector “for later” if the public product is deleted — delete or fully rehome; no zombie API.

## Verification

- `make verify` passes after the hard cut (codegen/OpenAPI/CLI docs/native tools/web/site included).
- Focused proof that kit extensions still work: `dev-cycle` enable exposes skills/loops/agents/tools; disable removes them from catalogs as today (plus any new package-owned kinds if added).
- Grep/contract gates: no remaining **living** public references to product Bundle APIs (`/api/bundles`, `compozy bundle`, `compozy__bundles_`, marketplace kind `bundle`) outside intentional historical changelog/archived notes.
- Homonym smoke: support bundle create/status/download still works; skill `--source bundled` / SourceBundled precedence unchanged.
- Unit/integration coverage for: extension-owned kit resource load on enable/disable; secrets/env missing reporting; inventory + preview; network confirm on extension path if applicable; provides allowlist; extension update reconcile/drift.
- Web: marketplace no longer offers bundle activate; extension detail shows kit contents / live inventory; shared marketplace/MCP/extension shells still work after kind-bundle strip.
- Playwright/e2e: bundle journeys removed or replaced with extension-only kit flows.
- QA: rewrite or delete `ET-024`–`030`, `ET-web-bundle-*`, related journeys/charters/seeds; flag user-visible behavior changes per `docs/qa/scenarios/` policy.
- Official skill + agent instruction files updated; no live lab processes left running after any QA.

## Removal inventory (product Extension Bundle only)

Homonyms out of scope (must survive): support diagnostics bundles; skill SourceBundled / `bundled` precedence; task ContextBundle; crash_bundle_path; CLI `outputBundle` helpers; sigstore/JS “bundle” wording.

### Runtime Go — delete or fully unwire

- Package `internal/bundles/**` (entire package).
- Extension: `internal/extension/bundle*.go`, `resources.bundles` in `manifest.go`, surface families `bundles` / `bundle.activation` in `surfaces/registry.go`, `ErrExtensionHasActiveBundles` in registry lifecycle, describe/manager snapshot fields for bundles.
- Daemon: `boot_automation_bundles.go`, `bundle_resources*.go`, Bundles deps in boot/runtime/server_options/extensions wiring, `native_bundle_resource_tools.go`, error maps for active-bundles / network-confirm.
- Contract: `internal/api/contract/bundles.go`; `MarketplaceKindBundle`; `ExtensionBundleSummaryPayload`; skill `InstalledFromBundle` provenance fields if only used for bundle ownership.
- API core: `internal/api/core/bundles.go`, `marketplace_bundles.go`, marketplace routes for bundle activations, resource mutate blocks for `bundle.activation`, network status merge of declared bundle channels.
- Spec/OpenAPI: `internal/api/spec/registry_bundles.go` + registration in `operation_registry.go` / `spec.go`.
- HTTP/UDS: `/api/bundles/catalog|preview|activations|activations/:id|network/settings`.
- CLI: `internal/cli/bundle*.go`, `newBundleCommand` in `root.go`, client methods for `/api/bundles/*`.
- Tools: `compozy__bundles_{list,info,activate,deactivate,status}`, toolset `compozy__bundles`, descriptors, toolmeta, `native-tool-catalog.json` rows.
- Resources: projector kinds `bundle` + `bundle.activation`.
- Store: `global_db_bundles.go`, sqlc `ListBundleActivationResourceSpecs` (kinds on `resource_records`).
- Automation/skills: package-source / `InstalledFromBundle` paths that exist only for bundle activation ownership.
- Mage: `magefiles/boundaries.go` marketplace↛bundles rule (remove with package).
- Agent docs: `AGENTS.md` / `CLAUDE.md` / `internal/AGENTS.md` living “bundles” checklist lines.

### Codegen / generated artifacts (must regenerate clean)

- `openapi/compozy.json` (tag `bundles`, eight ops).
- `web/src/generated/compozy-openapi.d.ts`.
- `packages/site/content/docs/api/bundles.mdx`.
- `packages/site/content/docs/cli/bundle/**`.
- Nav meta: `docs-navigation.ts`, `api/meta.json`, `cli/meta.json`, `resources/meta.json`, `docs-icons.ts`.

### Web — delete dedicated; strip shared

Delete: `bundle-activation-dialog*`, `bundle-activation-detail*`, routes `marketplace.bundles.tsx`, `marketplace.bundles.activations.$id.tsx`, related stories.
Strip (do not delete shared shells): marketplace kind `"bundle"`, preview/activate hooks/adapters, extensions activations API/hooks/query keys, OS marketplace window activation branch, mocks/fixtures, e2e bundle journey in `marketplace.spec.ts`, storybook route registry entries. Shared MCP/extension/skill marketplace components must keep working.

### Docs site — product Bundle teaching

Remove/rewrite: `resources/bundles.mdx`, `api/bundles.mdx`, `cli/bundle/*`, `extensions/install.mdx` and `develop.mdx` bundle sections, `marketplace/index.mdx` kind/activate teaching, `tools/toolsets.mdx` `compozy__bundles`, bridges docs that teach activation ownership, skills pages referencing `installed_from_bundle` / `capabilities-and-bundles.md`, `README.md` product “bundles” language.
Site tests asserting bundles nav/docs: `docs-discovery.test.ts`, `runtime-tools-canonical-docs.test.ts`, `runtime-docs-truth.test.ts`.

### Skills / QA / ecosystem / design / tasks

- `skills/compozy/references/capabilities-and-bundles.md`, `native-tools.md`, `tools-and-skills.md`, related SKILL.md routes — rewrite/delete product Bundle teaching.
- QA living contracts: `ET-024`–`ET-030`, `ET-web-bundle-*`, `NB-023`, `ET-020` (remove blocked by activations), marketplace/network journeys and charters that require bundle activate; seeds under `docs/qa/_seeds/**` that still prescribe `compozy bundle`.
- Keep as non-product: `ET-dev-cycle-skill-bundle.md` (SourceBundled skills), `MS-046`–`048` (support bundles).
- Ecosystem: archive/delete `docs/ecosystem/bundle-opportunities.md`; rewrite `docs/ecosystem/README.md` and `extension-opportunities.md` bundle framing.
- Memory checklists: `docs/_memory/standing_directives.md`, `spec-authoring-playbook.md` (remove living “bundles” surface).
- Design living refs: opendesign marketplace Bundles kind, `bundle-activation-detail.html`, site marketplace Bundles empty state.
- Active task trees that still specify product bundles: `.compozy/tasks/marketplace/**` (and product-bundle legs in `network-changes` / `agent-roles`) — archive note or rewrite so they are not living contracts.
- cy-web-docs-impact / cy-spec-preflight skill triggers listing `internal/bundles/**` / bundles surface — update.

### Tests that must disappear or be rewritten

- All `internal/bundles/*_test.go`.
- `internal/extension/bundle_*_test.go`, `registry_bundles_test.go`.
- CLI `bundle_test.go`; API/core marketplace/resources/network cases for activations; daemon native_tools / native_marketplace bundle cases; tools builtin/toolmeta bundle assertions.
- Web Vitest: bundle-activation dialog store, marketplace/extensions bundle coverage.
- Playwright bundle acquisition journey in `marketplace.spec.ts`.
- Site docs tests asserting product Bundle pages/tools.

### Keep historical (annotate only; not living contract)

- `.codex/plans/2026-04-14-extension-bundles.md`, `2026-05-03-extension-bundle-agents.md`, marketplace ledgers.
- QA bugs/reports that document past bundle bugs.
- `.compozy/tasks/_archived/**` bundle reports and evidence.

## References

- `docs/ecosystem/bundle-opportunities.md` — no first-party bundle ships; projection vs evolution gaps
- `docs/ecosystem/extension-opportunities.md` — extension-side framing
- `docs/ecosystem/README.md` — outcome-bundle model language
- `packages/site/content/docs/resources/bundles.mdx`
- `packages/site/content/docs/extensions/install.mdx`
- `packages/site/content/docs/extensions/develop.mdx`
- `internal/extension/manifest.go` — `ResourcesConfig`
- `internal/extension/bundle.go` — bundle/profile schema
- `internal/bundles/` — activation projector
- `internal/bundles/service_materialize_bridge.go` — disabled bridge projection
- `internal/daemon/boot_automation_bundles.go`, `bundle_resources*.go`, `native_bundle_resource_tools.go`
- `internal/api/contract/bundles.go`, `internal/api/core/bundles.go`, `marketplace_bundles.go`
- `internal/api/spec/registry_bundles.go`
- `internal/cli/bundle.go`
- `internal/tools/builtin_ids.go`, `internal/tools/builtin/bundles_resources.go`
- `openapi/compozy.json` — `/api/bundles/*`
- `web/src/routes/_app/marketplace.bundles.tsx`, `marketplace.bundles.activations.$id.tsx`
- `web/src/systems/marketplace/components/bundle-activation-dialog.tsx`
- `web/src/systems/extensions/components/bundle-activation-detail.tsx`
- `web/e2e/__tests__/marketplace.spec.ts`
- `extensions/dev-cycle/` — reference kit without bundles
- `extensions/bridges/` — bridge adapter extensions
- `skills/compozy/references/capabilities-and-bundles.md`
- `skills/compozy/references/native-tools.md`
- `docs/qa/scenarios/ET-024.md` … `ET-030.md`, `ET-web-bundle-*.md`
- `AGENTS.md`, `CLAUDE.md`, `internal/AGENTS.md`
