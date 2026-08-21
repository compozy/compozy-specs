---
status: pending
title: Session injection suppression and command catalog projection
type: backend
complexity: high
---

# Task 5: Session injection suppression and command catalog projection

## Overview

Delivers the session-facing half of the feature: provider-aware injection suppression (skills the session's provider already loads natively are omitted from prompt sections A and B — and nothing else), and the pre-overlay candidate projection that keeps every same-named skill invocable through qualified forms with `RootID` identity. This is Pedro's headline outcome — `/` in the session textbox lists skills from every enabled source with origin labels, while context never carries the same skill twice.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — no compat shims, no fallbacks, no placeholders (greenfield hard cuts)
</critical>

<requirements>
1. MUST plumb `Provider` into the harness session input (from `StartupPromptContext.Provider` / session `Info.Provider`) and compute `resolveSessionNativeSkillRoots` once per session: canonical provider ids (`claude`, `openclaw`, `hermes`; aliases `claude-code → claude`, `hermes-agent → hermes`; typed classification — never command-text or alias-format matching) × per-root preset knowledge × the session's effective env/home policy (`CLAUDE_CONFIG_DIR`, `HERMES_HOME`, `OPENCLAW_STATE_DIR`, `home_policy = isolated` relocate or eliminate native homes). Unknown/custom provider or unproven native loading → empty set → nothing suppressed (fail open).
2. MUST implement `SkillInjectionFilter` in the resolved harness policy: a skill is suppressed only when its WINNING root realpath is in the session's resolved native-root set; filtering applies to startup section A and per-turn catalog B only — command-catalog projection, settings/read APIs, enable/disable, shadows, and explicit `/<skill>` expansion are never filtered (Safety Invariant 6).
3. MUST filter BEFORE the per-turn augmenter's sha256 is computed (`prompt_skills.go` short-circuit) — enabling suppression changes the signature; the unchanged-marker still fires on repeat turns (UT-039 owns the ordering).
4. MUST log every suppression decision (`skills.injection.suppressed` — slog + harness observability tags `{session_id, skill, source slug, provider}`, log-only per the Monitoring section) so "why isn't this skill injected" is answerable from harness diagnostics.
5. MUST implement the pre-overlay candidate projection (ADR-016, B-005): candidates keyed by `RootID` + physical identity computed before the name-keyed overlay; the effective winner projects the bare command; every eligible same-named candidate projects its qualified command (`<slug>:<name>` display, `RootID` identity); expansion resolves the exact candidate by `RootID` and revalidates generation, source, and content — stale generations reject with the existing drift shape; collisions (skill vs builtin, skill vs skill) surface in diagnostics with qualified fallback, never silently dropped.
6. MUST carry origin labels on session command skill specs (catalog spec projection) so the web picker (task_06) renders chips without client-side derivation; compozy-native rows stay unlabeled.
7. MUST co-ship acpmock provider identities and matcher updates with the harness/prompt contract changes (L-007: E2E mock + matchers ship with the contract change; structured metadata, not rendered-prompt substrings).
8. MUST keep the invocation pipeline order intact (expand → augment) and leave `<invoked-skills>` expansion semantics unchanged apart from `RootID`-based candidate resolution.
</requirements>

## Subtasks

- [ ] 5.1 Harness `Provider` plumbing (`HarnessSessionInput` → resolved policy) from the session's resolved provider
- [ ] 5.2 `resolveSessionNativeSkillRoots`: canonical-id classifier + per-root reader knowledge + env/home-policy resolution + fail-open
- [ ] 5.3 `SkillInjectionFilter` applied to sections A and B; pre-hash placement in the augmenter
- [ ] 5.4 Suppression observability: slog decision log + harness tags + diagnostics surfacing
- [ ] 5.5 Pre-overlay candidate projection: `RootID`-keyed candidates, bare winner + qualified forms, custom-slug display stability
- [ ] 5.6 Expansion revalidation (generation/source/content) + collision diagnostics
- [ ] 5.7 Catalog spec origin labels for skill lanes
- [ ] 5.8 acpmock provider identities + matcher co-ship; e2e-runtime fixtures for the suppression matrix
- [ ] 5.9 Full assigned test suite (13 UT + 2 IT + 3 E2E)

## Implementation Details

Follow `_spec.md` Part II — System Architecture (Injection policy + Command catalog rows), Core Interfaces (harness block), Safety Invariants 6 and 7, and the Assumptions block (canonical provider vocabulary, per-root reader table: workspace `.agents/skills` → openclaw+hermes; global `~/.agents/skills` → openclaw only; both claude roots → claude). `internal/providerenv/env.go:283-292` `knownProviderHomeDirs` is the authority for provider-home layout under isolation — join it with the preset table rather than duplicating path math.

### Relevant Files

- `internal/daemon/harness_context.go:116-128,168-177,199-235,267` + `harness_context_session.go:9-18` + `harness_context_policy.go:3-53` — harness resolver gaining `Provider` + filter
- `internal/daemon/prompt_skills.go:18,40-160` — per-turn augmenter; sha256 short-circuit `catalogUnchanged:134-160` (filter goes before; LRU cap 2048)
- `internal/skills/catalog.go:14-30,46-138` — startup section A + catalog providers (filter hook), 200-char caps
- `internal/daemon/composed_assembler.go:35-46,302-308` — prompt-section consumption points
- `internal/providerenv/env.go:17-67,247-292` — home policy, `isolatedProviderHome`, `knownProviderHomeDirs` (native-home authority)
- `internal/config/provider.go:93-100` + `provider_effective.go:44-50` — `home_policy` (`operator`|`isolated`) resolution
- `internal/skills/registry_command.go:18-230` — `CommandCandidatesFor*`, `commandSkillSource:170-187` (origin labels), `commandSkillScope:203` — candidate projection seam
- `internal/daemon/session_commands.go:32-250` — `sessionCommandService` (`Catalog`, `Expand`, `project`, `commandSkillCandidates`, `sameSkillSource` drift check)
- `internal/command/catalog.go:20-107,145-230` — descriptors, lane order, `catalogRevision` (revision broadcast interacts with the task_02 generation fence)
- `internal/command/instructions.go:12-60` — `<invoked-skills>` expansion markers + budgets
- `internal/session/command_parser.go:10-31` + `manager_prompt_submit.go:103,172,293-309` — invocation pipeline order (expand → augment)
- `internal/session/command_catalog.go:13-103` — manager catalog + expansion entry
- `internal/acp/types.go:49-50` + `handlers_session_update.go:60-64` — `available_commands_update` (context: announced commands carry no source info — ADR-009 grounding)
- `internal/testutil/e2e` + acpmock fixtures — provider-identity mocks for IT-008/E2E-003

### Dependent Files

- `web/src/systems/session/hooks/use-session-commands.ts` + `web/src/components/assistant-ui/session-command-menu-model.ts` — consume origin labels (task_06)
- `internal/skills/registry.go` snapshot/generation surfaces — expansion revalidation reads task_02's fence

### Competitor References

- `.resources/claude-code/skills/loadSkillsDir.ts:67,216,270,340` — native skill loading, `userInvocable` default, `skillRoot` internals that never cross the SDK boundary (grounds ADR-009: the protocol cannot drive dedup)
- `.resources/claude-code/commands.ts:728` — description-string origin decoration (why announced commands are not a dedup contract)
- `.resources/openclaw/src/acp/commands.ts:1-50` — hardcoded ACP built-ins, zero skill announcement (fail-open justification)
- `.resources/hermes/acp_adapter/server.py:581-621,2068-2109` — hardcoded advertised commands; Hermes global home is `~/.hermes/skills` (per-root asymmetry evidence)

### Related ADRs

- [ADR-009](adrs/adr-009.md) — injection-only suppression via provider→native-roots mapping · [ADR-010](adrs/adr-010.md) — automatic, no config key · [ADR-013](adrs/adr-013.md) — origin labels where skills are chosen · [ADR-016](adrs/adr-016.md) — pre-overlay candidate projection + RootID

### Web/Docs Impact

- `web/`: no component edits here — the picker consumes the catalog spec origin labels in task_06 (`session-command-menu-model.ts:242-249` trailing slot). Session command payload additions regenerate types via `make codegen` if the wire spec changes.
- `packages/site`: none in this task — suppression + qualified-invocation docs land in task_07 (`content/docs/skills/index.mdx`).
- QA impact: flag (walk deferred to task_09) one new content-addressed `untested` scenario: session picker + suppression — absorbed skills listed with origin, qualified homonym invocation, explicit `/` invocation of a suppressed skill still injects, per-provider suppression matrix visible in harness diagnostics, source-disabled drift rejection.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — hooks/MCP declarations on absorbed skills follow existing user-tier rules untouched (checked: `metadata.compozy.*` pipeline, extension surfaces, bridge SDKs). acpmock co-ship is test infrastructure, not a public surface.
- Agent manageability: suppression decisions observable via harness diagnostics/tags; command catalog + expansion remain the structured session surfaces (shapes unchanged, additive origin label); drift rejection keeps the existing deterministic error shape.
- Config lifecycle: none — suppression is automatic by design (ADR-010 explicitly rejects a config key). Checked: no `config.toml` surface.

## Deliverables

- Session-resolved native-root suppression with fail-open semantics, filtering sections A+B only, pre-hash
- Suppression decision observability (slog + tags + diagnostics)
- Pre-overlay candidate projection with bare winner + qualified forms, `RootID` expansion revalidation, collision diagnostics
- Catalog origin labels for skill rows; acpmock/matcher co-ship
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there. Owning suites: harness/prompt suites under `internal/daemon`, `internal/skills/catalog_test.go`, `internal/skills/registry_command` coverage, session-command suites, e2e runtime lane (acpmock).

- [ ] UT-035, UT-036, UT-037, UT-038, UT-040, UT-076 — suppression matrix (claude/openclaw/hermes per-root, winner-key, unknown provider, custom origin, observability)
- [ ] UT-039 — filter-before-sha256 ordering (short-circuit correctness)
- [ ] UT-085 — session-native-root resolution (aliases, env overrides, isolated home, typed classification)
- [ ] UT-041, UT-042, UT-043, UT-044, UT-081 — qualified descriptors, source identity stability, collision diagnostics, drift rejection, pre-overlay projection by RootID
- [ ] IT-008 — suppression matrix through real prompt assembly (owning case for ADR-009)
- [ ] IT-010 — source disabled mid-session (transcript intact, next-turn catalog + picker drop)
- [ ] E2E-002 — picker → invocation with verified content; qualified homonym; verification-failure rejection
- [ ] E2E-003 — per-provider injected-catalog exclusion with commands endpoint unaffected
- [ ] E2E-004 — pick → disable source → submit → drift rejection

## Success Criteria

- Every assigned test case implemented and passing; `make gate` green (task close); e2e-runtime lane green
- Suppression provably never removes a skill from picker/lists/management or blocks explicit invocation (IT-008 assertions)
- Repeat turns with unchanged filtered catalogs still hit the unchanged-marker short-circuit (no token regression)
- Shadowed homonyms remain invocable via qualified forms resolved by `RootID`, surviving display-slug re-suffixing
