---
status: completed
title: "Distribution: source union, gitsrc, sidecars, publish, search, update (Phase E)"
type: backend
complexity: high
---

# Task 5: Distribution — source union, gitsrc, sidecars, publish, search, update (Phase E)

## Overview

Open distribution without a gatekeeper (R2/R3/R10, ADR-005): the install request hard-cuts to the `curated|github|git|local_path` union, a new git source and GitHub digest sidecars land (integrity-only — never trust elevation), `compozy extension publish` creates releases with server-side credentials, search fans out across sources, and batch update reports per-item progress. This task also completes the native-tool set (nine) and the transport-parity + secret-hygiene proof across the whole surface.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST hard-cut `contract.InstallExtensionRequest` to the source union (no `dev` member — dev is a lifecycle overlay); the CLI-owned shorthand parser maps `github:owner/repo[@ref]`, `git:<url>[@ref]`, filesystem paths, and bare `owner/repo` (curated-then-github plan) — a nonexistent `./path` errors naming the path, never degrading to a slug lookup.
- MUST implement `internal/registry/gitsrc` (shallow clone at ref via `os/exec` with context timeout; `Search` returns `ErrNotSupported`; absent binary → deterministic `ErrGitUnavailable`) and register it in the daemon source loader.
- MUST verify archive digests before any registry write with zero partial state on failure (invariant 4, `rollbackFailedInstall` extended to all sources); digest sidecars are integrity-only: `digest_matched` never elevates tier/`checksum_verified`/badges/consent; mismatch aborts with no consent override (invariant 5).
- MUST implement `PublishRelease` with no credential field on any request/tool input: the GitHub credential resolves server-side (daemon env/vault binding) for the native tool and from the CLI process env for the local verb; it appears in zero bytes of payloads, errors, events, transcripts, or logs (invariant 13).
- MUST implement search union (`GET /api/extensions/search?q=&sources=&limit=&cursor=`) composing curated store + GitHub topic search in the daemon service layer (never in `internal/marketplace`), each result tagged source/tier/integrity; update discovery stays projection-only with the N-004 degradation contract (2s per-source bound, 15m cache, `sources_degraded` marker).
- MUST surface `update --all` per-item partial progress from the daemon `UpdateBatch` through HTTP and UDS; provenance `git_url`/github becomes truthful; unverified side-load collapses to policy + one consent (`--allow-unverified` IS the consent).
- MUST complete `ExtensionStatusCode` (UT-063 table: dev-link 409s, generation 400, validation 400 + issues, bridge.adapter 400, git-unavailable 503-class) and the nine-tool native set (`search`, `provenance`, `publish` join; `publish` = `open_world` + interaction-gated) with structured outputs.
- MUST complete the `ExtensionPayload` contract additions in this task (`update_available`, `remote_version`, `consecutive_failures`, `restart_backoff`, `digest_matched`, plus task_04's dev fields already landed) so downstream consumers (task_07 CLI rendering, task_08 web) depend only on task_05.
- MUST prove transport parity (IT-010) and cross-transport secret absence (IT-016) over the complete surface.
</requirements>

## Subtasks

- [x] 5.1 Contract hard cut + CLI shorthand parser (`internal/cli/extension_install_parse.go`) + web-visible request shape via codegen
- [x] 5.2 `internal/registry/gitsrc` + boundaries registration + daemon source-loader growth
- [x] 5.3 Digest sidecar fetch/verify (integrity-only provenance facts) + rollback-on-failure across all sources
- [x] 5.4 `internal/extension/publish.go` + release upload in `internal/registry/github` + credential-binding resolution
- [x] 5.5 Search union service + route/UDS + cursor pagination; update projection degradation contract
- [x] 5.6 `UpdateBatch` per-item surfacing; provenance truth; trust-affordance collapse (policy + consent)
- [x] 5.7 Status-code table completion + native tools to nine + toolset description truth
- [x] 5.8 Implement every assigned test case incl. the two journeys (E2E-002 distribution, E2E-003 agent-native full loop)

## Implementation Details

TechSpec: Core Interfaces (Distribution block), Data Models (InstallExtensionRequest), API Endpoints, Integration Points (GitHub/git), Safety Invariants 4–5, 12–13, Technical Considerations (update discovery). Phase E gate: mocked-GitHub integration + e2e publish→install.

### Relevant Files

- `internal/api/contract` + `openapi/compozy.json` + `web/src/generated/compozy-openapi.d.ts` — request hard cut co-ship (L-007)
- `internal/registry/{source.go,github/}` — Source interface + existing client (release resolve, asset download) gaining sidecar fetch + release create/upload
- `internal/extension/{marketplace_trust.go,marketplace_lifecycle.go,provenance.go,registry_install.go}` — trust collapse, install pipelines, UpdateBatch, provenance truth (suite: `marketplace_lifecycle_test.go`)
- `internal/daemon/extensions.go` — search-union composition + source loader (SD-008)
- `internal/cli/{extension.go,extension_marketplace.go,marketplace.go}` — install/update/search/publish verbs
- `internal/daemon/native_extension_tools.go` (+ `native_extension_tools_integration_test.go`) — nine-tool set + risk classes (`extensionToolBindings` L63 registration point; `defaultDaemonExtensionMarketplaceSourceLoader` L377 grows the git source)
- `internal/daemon/{extensions.go,extensions_update_batch.go}` — `Install`/`Update`/`UpdateBatch`/`rollbackFailedInstall` service methods this task extends
- `internal/testutil/e2e/bridges_extensions.go` — `InstallExtension` harness helper for the E2E-002/003 journeys

### Dependent Files

- `web/src/systems/marketplace/*` — install forms + badges consume the union (task_08)
- `internal/api/core/extensions.go` — status-code table + parity suite home

### Related ADRs

- [ADR-005: GitHub/git-first distribution](adrs/adr-005.md) — primary
- [ADR-002: First-class dev lane](adrs/adr-002.md) — trust-affordance collapse on the published lane
- [ADR-006: Closed-surface positioning](adrs/adr-006.md) — bridge.adapter rejection stays deterministic

### Competitor References

- `.resources/claude-code/utils/plugins/schemas.ts:906-1000` + `.resources/claude-code/utils/plugins/parseMarketplaceInput.ts` — source-union input parsing precedent
- `.resources/openclaw/docs/tools/clawhub.md` — layered distribution (hub + npm + URLs)
- `.resources/pi/packages/coding-agent/docs/packages.md:20-32` — npm/git/URL/path install grammar with pins

## Web/Docs Impact

Request-shape and payload changes reach `web/` through regenerated types (co-shipped here with E2E mocks per L-007); the marketplace forms/badges/update surfaces that consume them are task_08. Publish guide + install docs owned by task_09.

**QA impact**: reset the ET-015..ET-023 install/update scenarios touching install sources, trust prompts, or update flows to `untested`; add content-addressed `untested` scenarios for `github:`/`git:` installs, publish→install round trip, and batch-update partial progress.

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: registries grow a source kind (`gitsrc`) + sidecar convention; curated feed unchanged; bundles/skills pipelines untouched (skills stay on ClawHub — ADR-005).
- Agent Manageability: the complete nine-tool native set with structured outputs and deterministic errors; agents run the full loop with no shell-outs (E2E-003 proves it); HTTP/UDS parity asserted (IT-010).
- Config Lifecycle: consumes `[extensions.trust]`/`[extensions.sources.*]` (defined task_02); no new keys — checked: validation/examples unchanged beyond task_02's cut.

## Deliverables

- All four install sources one-command; publish→discover→install works against mocks with zero Compozy-side gatekeeping
- Integrity facts never masquerade as trust; secrets cross no transport
- Nine native tools; complete status-code map
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] UT-022, UT-023, UT-024, UT-025 — dev-lane zero trust evals; consent/policy matrix; rollback on mismatch
- [x] UT-026, UT-027, UT-028, UT-029, UT-030 — shorthand parser union (order-asserted curated-then-github plan)
- [x] UT-031, UT-032 — gitsrc clone argv + missing-binary diagnostic
- [x] UT-033, UT-034 — publish upload + sidecar; unresolvable credential binding
- [x] UT-063 — full `ExtensionStatusCode` table
- [x] UT-068, UT-069 — sidecar never elevates trust; bridge.adapter rejection
- [x] IT-002, IT-003, IT-004 — install pipelines per source (local, mocked GitHub ± sidecar, git fixture repo)
- [x] IT-007 — `update --all` per-item outcomes over HTTP and UDS
- [x] IT-008 — search union merge + badges + stable cursor
- [x] IT-010 — HTTP↔UDS parity (dev/reload/logs/search)
- [x] IT-011 — all nine native tools invoked with risk registration asserted
- [x] IT-016 — secret absence across every transport surface end to end
- [x] E2E-002 — publish → second home install → verify → update → remove
- [x] E2E-003 — agent completes the FULL authoring loop via native tools only, gates approved in-band

Contract co-ship gates (L-007): `make codegen && make codegen-check` after the request hard cut + payload completion + `bunx turbo run typecheck test --filter=./web` (regenerated types compile; E2E mocks updated in the same change).

## Success Criteria

- Every assigned test case implemented and passing
- Own-code install ≤1 command + ≤1 consent; no marketplace vocabulary in dev/local errors (brief R2 measured)
- Raw credentials appear in zero bytes of any transport (IT-016 green)
- Listing never blocks on a dead source (degradation contract honored)

## Completion Notes

Implemented the complete published-distribution lane across registry sources, runtime, transports,
CLI, native tools, generated contracts, Web consumers, and the official Compozy skill. Install now
uses the `curated|github|git|local_path` source union; Git sources clone with the system `git` binary;
GitHub releases use SHA-256 sidecars as integrity evidence only; and publish credentials resolve at
the execution boundary without entering request payloads. Search merges curated and GitHub results
behind bounded source calls, a 15-minute cache, stable cursors, and explicit degradation metadata.
Batch update preserves every per-item outcome, and CLI search now preserves the complete page
contract instead of dropping `next_cursor` and `sources_degraded`.

Library research considered `google/go-github` and `go-git`. Neither was adopted: the existing
GitHub client already owns the typed protocol, retry, timeout, and credential boundary, while the
accepted git contract explicitly requires the system CLI and its deterministic unavailable-binary
error. Existing `Masterminds/semver` and the shared deterministic tar-gzip helper cover the remaining
generation needs without a second dependency stack.

Fresh focused evidence after source freeze:

- `make lint` — PASS, zero issues; source-size and source-policy gates PASS.
- `make codegen-check` — PASS; all generated contracts and 13 extension manifests are current.
- Race tests over 14 affected Go package groups — 5,219 tests PASS.
- Fresh `go test -race ./internal/cli -count=1` — 1,272 tests PASS after the search-page refactor.
- Tagged distribution, secret-hygiene, and both E2E journeys — 8 tests PASS.
- Forced Web Turbo lane — 5/5 tasks, 515 files and 4,046 tests PASS; lint and typecheck clean.
- `git diff --check` — PASS; QA tracker materialized with 663 scenarios.
- Official skill startup prompt — 31,978/32,000 runes, leaving 22 runes of headroom.

The single program-wide `make verify` remains deferred until all eleven tasks are complete, per the
accepted loop execution contract.

Compozy Impact Audit:

- Native tools: added `compozy__extensions_search`, `compozy__extensions_provenance`, and `compozy__extensions_publish`; updated install/update/list/info/remove schemas, descriptors, risk flags, availability wiring, catalog golden coverage, and native lifecycle tests. Publish is `open_world` and interaction-gated; discovery and provenance remain structured read paths.
- Extensibility and hooks: added the git registry source, GitHub sidecar/publish/search behaviors, source-union loading, cached degradation-aware discovery, and batch update projection. The task consumes Task 02 trust/source configuration without adding keys. Checked extension hooks, skills/capabilities, bundles, bridge SDKs, MCP sidecars, and resources; their contracts are unchanged.
- Workspace data isolation: published installs, public discovery cache, and provenance are global distribution data; dev overlays remain workspace-scoped. Native list/info/remove resolve the trusted workspace server-side, and hostile transport/E2E coverage proves requests cannot select another workspace. Search cache contains public source projections only and carries no workspace/session/agent data.
- Official Compozy skill: updated `skills/compozy/references/capabilities-and-bundles.md`, `native-tools.md`, and `tools-and-skills.md` with the source grammar, integrity/trust distinction, publish credential boundary, search degradation, provenance, batch update, and native-tool guidance.

Web/Docs Impact:

- `web/`: regenerated OpenAPI types and hard-cut five marketplace fixtures/adapters/hooks/controllers to the source-union and current payload contract. Task 08 still owns the visible install/update UI.
- `packages/site`: no public guide changed; Task 09 owns publish/install documentation after the distribution surface settles.
- QA: added content-addressed `untested` scenarios for published-source installs, publish/install round trip, and batch-update partial progress. ET-015..ET-023 were already `untested`, so no stale verdict required resetting.
- Visual verification: not applicable in this task; generated Web consumers changed, but no user-visible UI layout or styling changed.
