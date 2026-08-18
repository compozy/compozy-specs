---
status: pending
title: QA Plan and Session Charters
type: qa-report
complexity: high
---

# Task 05: QA Plan and Session Charters

## Overview

Plans the real-user QA cycle for the modal redesign against the living `docs/qa/` tree: journey updates, content-addressed scenario mint/reset for all 16 entity editors, and session charters covering foundation shell, low-risk dialogs, dense Simple/Advanced migrations, and provider auth gating.

<critical>
ALWAYS READ `_techspec.md`, every ADR (if any), `VISUAL-VALIDATION.md`, `STATE-MATRIX.md`, `CHECKLIST.md`, and every per-task memory file before planning.
</critical>

<requirements>
- MUST activate the `qa-report` skill with `qa-docs-path=docs/qa` (bootstrap the tree if absent).
- MUST update journey flowcharts in `docs/qa/journeys/` for operator flows that open the migrated dialogs (session start, workspace add, agent create, bridge setup, MCP/settings, vault, sandbox, knowledge, network channel, provider settings).
- MUST mint or reset scenario files in `docs/qa/scenarios/` so every user-visible change from tasks 01–04 is `untested` (or new content-addressed) — including Simple/Advanced disclosure, secret rotate/replace, ImmutableIdentity locks, auth_mode gating, and host-token shell chrome.
- MUST write session charters in `docs/qa/charters/` for this cycle: targeted tier covering dense dialogs + provider auth gate, plus one adjacent canary journey (e.g. marketplace MCP install sharing SecretField).
- MUST express coverage as scenario `entry_points` on journey-derived rows, not as standalone abstract test cases.
- MUST map regression hot spots from `_techspec.md` §5.2 / §6 / §14 into charter selection (T1 cut, D2 delivery fold, T3 auth gate, wizard collapse validation).
- SHOULD reuse existing scenario IDs when behavior matches (`MS-provider-detail-modal`, bridge/MCP/session scenarios) and only mint new content-addressed files for net-new behavior.
</requirements>

## Subtasks

- [ ] 5.1 Inventory scenarios touched by tasks 01–04 and mark `qa_status: untested` where behavior changed
- [ ] 5.2 Mint content-addressed scenarios for net-new Simple/Advanced + secret rotate paths lacking owners
- [ ] 5.3 Update journeys for OS dialog entry points
- [ ] 5.4 Author cycle charters (targeted + canary)
- [ ] 5.5 Cross-check every public surface from tasks 01–04 appears as an entry_point somewhere in the plan

## Implementation Details

Activate `qa-report`. Living tree only — never create a per-round `qa/` tree. TechSpec §10–11 and `_tasks.md` AGH Impact Audit define the flag set. Visual Contract evidence from tasks 01–04 is implementation evidence, not a substitute for scenario planning.

### Relevant Files

- `docs/qa/scenarios/` — especially provider, bridge, MCP, session, vault, workspace scenarios
- `docs/qa/journeys/`, `docs/qa/charters/`
- `_techspec.md` §4–6, §10–11
- `VISUAL-VALIDATION.md`, `STATE-MATRIX.md`

### Dependent Files

- task_06 execution consumes the charters and in-scope scenario set

## Deliverables

- Updated journeys + charters for this cycle
- Scenario mint/reset complete for tasks 01–04 user-visible behavior
- Charter selection cites TechSpec hot spots (T1/T3/D2/wizard collapse)

## Tests

- [ ] Every task_01–04 user-visible surface maps to at least one `docs/qa/scenarios/*.md` entry_point with `qa_status: untested` (or newly minted untested)
- [ ] Charters reference concrete scenario IDs and journey paths
- [ ] No orphan scenario that claims this wave without an owning journey entry_point

### Web/Docs Impact

- `web/`: none for this planning task — checked implementation already landed in 01–04.
- `packages/site`: none — checked docs/qa planning only.
- QA impact: this task *is* the flag/plan step — scenarios must be untested before task_06 runs.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — planning only.
- Agent manageability: include CLI/HTTP entry points in journeys where dialogs have agent-operable equivalents (create agent, bridge, vault) as canaries — no new verbs.
- Config lifecycle: none.

### AGH Impact Audit

- Native tools: no impact — planning only; checked no tool changes required for QA docs.
- Extensibility and hooks: no impact — planning only.
- Workspace data isolation: scenarios must include scoped create paths (workspace scope) where dialogs expose scope.
- Official AGH skill: no impact — planning only.

## Success Criteria

- Living `docs/qa/` plan ready for task_06 with untested in-scope scenarios and charters
- Hot spots T1/T3/D2/wizard collapse explicitly covered
