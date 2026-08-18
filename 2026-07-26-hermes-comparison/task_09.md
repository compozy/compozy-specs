---
status: pending
title: "Extensibility: registry cache, installable MCP catalog, parse snapshot, and when.* gates"
type: backend
complexity: high
---

# Task 9: Extensibility: registry cache, installable MCP catalog, parse snapshot, and when.* gates

## Overview

Registry/skills machinery — offline index-cache + embedded seed, installable-MCP catalog with
curated defaults, skill parse snapshot, and conditional `when.*` activation gates. Closes the four
Hermes product-mechanic gaps in AGH's extension surface (ADR-009): every search is a live network
round-trip; installing an MCP server is fully manual; cold starts reparse unchanged catalogs; and
every enabled skill is advertised regardless of platform/tool availability.

**Authoritative sources:** this suite has no `_prd.md` / `_user_stories.md` / `_tests.md`.
`_techspec.md` §3.9 and `adrs/adr-009.md` are authoritative. Concrete test cases are inline below
(exact input/condition/expected).

Merges former tasks 24+27+28+25. Former edge `24→27` is subtask order inside this slice (MCP
catalog rides the registry cache).

**CRITICAL subtask order:**
1. Registry cache + embedded seed (ex-24)
2. Installable-MCP catalog (ex-27 — rides cache/catalog machinery)
3. Skill parse snapshot (ex-28)
4. Conditional `when.*` gates (ex-25)

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
### Registry cache (ex-24)
1. MUST add a TTL-bounded catalog cache in front of `MultiRegistry`: hits serve offline; misses
   fetch and refresh; explicit refresh verb exists.
2. Cache keys MUST include source+query only — never workspace-scoped installed state (isolation
   rule from analysis 08 §5.4).
3. MUST embed a seed catalog (shipped in-binary) with an `official`/optional tier so search works
   on first boot offline.
4. MUST keep search result provenance truthful (cached-at timestamp surfaced; stale data labeled).

### Installable-MCP catalog (ex-27)
5. MUST add an `mcp` registry `PackageType` whose manifest declares transport, auth requirements,
   git-install source, and a curated default-tool subset.
6. Mutating tools MUST be opt-in via the existing risk-flag model (safe subset by default).
7. Installed MCP tools MUST flow through the existing MCP host (`internal/mcp/`) — never the
   native toolset (no `agh__*` minting).
8. Install/list/remove surfaces: CLI (`agh mcp install/...`) + HTTP/UDS parity + codegen co-ship;
   git-install honors symlink-escape hardening on materialized paths.

### Parse snapshot (ex-28)
9. MUST serialize the parsed skill catalog keyed by the existing `filesnap` manifest; snapshot hit
   skips reparse, any source change invalidates.
10. MUST NOT bypass the load-time security scan for non-bundled skills (`VerifyContent` runs on
    every load regardless of snapshot — the scan invariant is not cacheable).
11. Corrupt/incompatible snapshot MUST fall back to full reparse silently-with-log (fail-open).

### when.* gates (ex-25)
12. MUST extend `metadata.agh` with `when.{platforms,environments,requires_tools,
    requires_capabilities}`; gates evaluate at catalog build per agent/workspace.
13. Gated-out skills MUST NOT be advertised to the agent; their state is discoverable (list shows
    "inactive: gate X unmet") — truthful, not hidden.
14. MUST keep evaluation deterministic and cheap (no network at catalog build).
15. MUST document the new frontmatter in `skills/agh/` references + site docs.
</requirements>

## Subtasks (order: cache → MCP catalog → parse snapshot → when.*)

- [ ] 9.1 Cache layer (TTL, refresh verb, offline path) + provenance fields.
- [ ] 9.2 Embedded seed catalog + `official` tier + CLI/HTTP/UDS search flags (`--offline`,
      `--refresh`) + docs.
- [ ] 9.3 `mcp` PackageType + manifest schema + validation.
- [ ] 9.4 Install pipeline (git fetch, materialization, hardening) + MCP host registration +
      default-subset curation via risk flags; CLI/HTTP/UDS surfaces + docs.
- [ ] 9.5 Snapshot serialize/load + invalidation via filesnap + fail-open fallback + version
      stamping + cold-start benchmark before/after.
- [ ] 9.6 Frontmatter schema + parser (allowlisted keys, unknown-key lint) + gate evaluation in
      catalog build + inactive-state surfacing + docs + bundled-skill audit.

## Implementation Details

See `_techspec.md` §3.9 / ADR-009. Two-touch note (ADR-009 §5): these touch distinct packages
(`internal/registry`, `internal/skills`) — do not fold more than two into one patch series.
Trust posture: source-tier ceilings, never hardcoded repo allow-lists. Non-goals: in-process
extension execution, untyped hooks, web/UI extension tabs, skill→automation materialization.

### Relevant Files

- `internal/registry/` — cache + seed + `mcp` PackageType + manifest
- `internal/mcp/` — registration of installed servers
- `internal/skills/` — parse snapshot + frontmatter schema + `registry_agent` gate evaluation
- `internal/filesnap/` — snapshot keying (consumed)
- embed home for the seed catalog

### Dependent Files

- `internal/cli/` — search flags + `agh mcp install/...` verbs
- `internal/api/contract/` + TS — catalog/install payloads
- `skills/agh/` — registry usage, MCP install, `when.*` frontmatter docs
- `packages/site` — frontmatter / MCP catalog docs
- skill list surfaces (CLI/HTTP) — inactive-state field

### Related ADRs

- [ADR-009: Extensibility batch — offline registry cache, conditional skill activation,
  installable-MCP catalog, parse snapshot](adrs/adr-009.md) — all four units; sequencing;
  non-goals

### Competitor References

- `.resources/hermes/skills/index-cache/`, `.resources/hermes/optional-skills/` — cache + optional
  distribution model
- `.resources/hermes/optional-mcps/` — manifest + curation model
- Hermes conditional activation per `analysis/08_analysis_skills-plugins.md` §3.4

## Deliverables

- Offline-capable, fast registry search with truthful staleness + embedded seed
- One-command MCP install with safe-by-default tool exposure
- Cold-start reparse skipped on unchanged catalogs (scan-on-every-load intact)
- Gated skill advertisement with truthful inactive states
- Every test case in `## Tests` implemented and passing **(REQUIRED)**

## Tests

No `_tests.md` for this suite — concrete inline cases below. Coverage target ≥80% on touched
packages; all Go tests under `-race`.

### Registry cache (ex-24)

- Unit (`internal/registry/*_test.go` — extend):
  - [ ] Cache hit within TTL → zero network calls (fake transport assertion)
  - [ ] Expired TTL → refetch + cache update; network down → stale served with staleness marker
  - [ ] First boot, network down → seed catalog serves official-tier results
  - [ ] Cache key excludes workspace state (two workspaces share cache entries for same query)
- Integration (`make test-integration`):
  - [ ] `agh skill search` offline round-trip against the seed; `--refresh` forces fetch
- E2E: N/A — CLI/registry surface fully covered above; no web lane in this half

### Installable-MCP catalog (ex-27)

- Unit (`internal/registry/*_test.go` + `internal/mcp/*_test.go` — extend):
  - [ ] Manifest with mutating tools → default subset excludes them; opt-in flag includes them
  - [ ] Invalid manifest (missing transport/auth fields) → deterministic validation error
  - [ ] Git-install materialization with a symlink escaping the install root → rejected
  - [ ] Installed server's tools appear via the MCP host, never in the native `agh__*` namespace
- Integration (`make test-integration`):
  - [ ] Catalog entry → `agh mcp install` → server spawns → curated tools callable in a session;
        remove → gone without daemon restart
- E2E: N/A — operator install flow covered by integration; W7 QA includes an install scenario

### Parse snapshot (ex-28)

- Unit (`internal/skills/*_test.go` loader suite — extend):
  - [ ] Unchanged sources → snapshot hit, parse functions not invoked (spy assertion)
  - [ ] One skill file modified → that scope invalidated and reparsed
  - [ ] Corrupt snapshot bytes → full reparse, no error surfaced to boot
  - [ ] Security scan invoked on every load even on snapshot hit (non-bundled fixture)
- Integration (`make test-integration`):
  - [ ] Daemon restart with unchanged catalog → identical catalog content served (deep-equal)
- E2E: N/A — boot-path performance internals; no behavioral surface

### when.* gates (ex-25)

- Unit (`internal/skills/*_test.go` catalog/registry suites — extend):
  - [ ] `when.platforms: [linux]` on darwin build fixture → skill absent from agent catalog,
        present in list as inactive-with-reason
  - [ ] `requires_tools` naming an unavailable tool → gated out; tool becomes available → active
        on next build
  - [ ] Unknown `when.*` key → lint/parse error (allowlist enforcement)
  - [ ] No `when` block → behavior identical to today (backward-clean, zero-legacy: no compat
        shim needed)
- Integration (`make test-integration`):
  - [ ] Agent session prompt assembly excludes gated skills (token-level assertion on the
        advertised set)
- E2E: N/A — catalog semantics fully covered above; skill list UI unchanged in shape (inactive
  badge covered by web unit test if the view changes)

## Success Criteria

- Every assigned test case implemented and passing
- Coverage ≥80% on touched packages
- Search latency on cache hit measured and reported (notes); offline search functional
- Install-to-usable measured in one command + zero manual config edits; mutating tools unexposed
  without opt-in
- Cold-start catalog build time reduction measured; scan-on-every-load invariant proven intact
- No skill is offered whose declared requirements are unmet (truthful-UI check)
- Advertised-skill token count drops on the gated fixture (measured in notes)
