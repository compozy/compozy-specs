---
status: completed
title: Phase 2: compatible-clients listing handoff
type: chore
complexity: low
---

# Task 9: Phase 2: compatible-clients listing handoff

## Overview

Closes the evidence handoff for a user-owned PR to `agentplugins/agent-plugins-site`. Task 08 proved the 8-item conformance checklist and complete Claude Code/Hermes delivery while recording OpenClaw's per-session MCP limitation. The external claim follows that evidence and must wait until the CompozyOS interoperability page is deployed.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST verify the evidence gate: task_08's dated report exists with the 8-item conformance walk, provider matrix, and current full-gate record.
2. MUST record the current contribution shape of `agentplugins/agent-plugins-site`: client name Compozy, skills ✓, MCP transports `stdio, streamable-http` only, official light/dark logos, and the deployed interoperability URL.
3. MUST preserve the evidence boundary: Claude Code and Hermes passed the complete provider walk; OpenClaw's ACP bridge is not part of the portable MCP delivery claim.
4. MUST leave fork, branch, and PR creation to the user, as explicitly requested on 2026-08-16.
5. MUST apply `COPY.md` claim standards to the handoff facts; the eventual external PR remains user-owned.
</requirements>

## Subtasks

- [x] 9.1 Evidence gate checked: 8/8 conformance, Claude Code/Hermes pass, OpenClaw limitation recorded
- [x] 9.2 Current upstream entry/logo shape and merged NanoClaw PR #50 precedent verified
- [x] 9.3 Canonical spec, public docs, official skill, and QA verdict narrowed to the evidenced claim
- [x] 9.4 External fork/branch/PR left untouched for the user to open after docs deployment

## Implementation Details

External repo: `github.com/agentplugins/agent-plugins-site`. The current entry shape was verified against `lib/compatible-clients.ts`; merged NanoClaw PR #50 is the accepted precedent. Instructions URL: `https://compozy.com/docs/extensions/agent-plugins` after this CompozyOS PR is merged and deployed. Evidence source: `docs/qa/reports/2026-08-16-agent-plugins.md`.

### Relevant Files

- `.compozy/tasks/agent-plugins/analysis/analysis_agent_plugins_standard.md` §F — the compatible-clients table shape + PR precedent
- `docs/qa/reports/` — evidence to cite

### Dependent Files

- `packages/site/content/docs/extensions/agent-plugins.mdx` — must be deployed before the external listing.

### Related ADRs

- [ADR-001](adrs/adr-001.md) — Phase 2 scope definition

### Web/Docs Impact

- `web/`: no impact — checked marketplace and extension surfaces; runtime behavior is unchanged.
- `packages/site`: provider-delivery boundary added to the interoperability page.
- QA impact: existing provider-delivery scenario now passes the narrowed contract using the already-recorded matrix; no runtime re-walk is required because behavior did not change.

### Extensibility / Agent Manageability / Config Lifecycle

- No runtime, extension-hook, native-tool, workspace-isolation, or `config.toml` change. The contract now matches the existing provider capability gate.

## Deliverables

- Evidence and upstream-shape handoff for the user-owned external PR
- Deployed-doc prerequisite recorded
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED — none assigned; external contribution)**

## Tests

Cases assigned from `_tests.md`: none — the evidence this PR cites was produced by the runtime distribution journey (owned by task_03) and task_08's recorded walks.

## Success Criteria

- Evidence gate is verifiably satisfied for the narrowed claim
- Handoff records the exact transports, instructions URL, logo requirement, and accepted precedent
- No external branch or PR was created; the user retains ownership of that action

## Completion Notes

- User decision: narrow the provider claim to Claude Code and Hermes; record OpenClaw's ACP limitation.
- CompozyOS delivery: PR [compozy/compozy#419](https://github.com/compozy/compozy/pull/419) opened with the full QA report and hosted browser evidence.
- External action: deferred to the user. No fork, branch, commit, or PR was created in `agentplugins/agent-plugins-site`.
- Required order: merge and deploy the CompozyOS PR, confirm the interoperability URL returns 200, then open the external listing.
