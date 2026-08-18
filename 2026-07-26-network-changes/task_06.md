---
status: pending
title: "Docs, official skill, glossary & delete-target sweep"
type: docs
complexity: medium
---

# Task 6: Docs, official skill, glossary & delete-target sweep

## Overview

Hard-cut product language and agent guidance to Local/Live opt-in participation: rewrite site network/autonomy pages, regenerate CLI/config references, rewrite the official AGH skill network/orchestration/tooling guides, update glossary/COPY, flag QA tracker rows, and run a repository-wide delete-target search gate plus final `make verify`. This closes Build Order step 9 and is the last implementation task before the QA pair.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. `packages/site` network + autonomy pages MUST delete "bind always" / every-run-channel doctrine and teach Local default + Live opt-in + orchestration independence.
2. Config reference MUST document `[network.live.defaults|limits]`, kept keys, and removed keys as rejected; protocol/NATS broker-subject pages MUST be retired or rewritten for in-process delivery (ADR-010).
3. Official skill `skills/agh/references/network.md`, `tasks-and-orchestration.md`, and `native-tools.md` MUST describe modes/admission/bounds/usage and gated coordination tooling.
4. Glossary Coordination Channel entry + COPY.md mode naming MUST match the amended PRD (Local/Live); no mailbox public vocabulary.
5. Repository-wide delete-target search MUST return zero hits for TechSpec Delete Targets symbols/fields/keys/doctrine strings (or only intentional historical research under `.compozy/tasks/network-changes/` research docs).
6. `docs/qa/state.csv` MUST add new `untested` rows for participation controls, coordination toggle+invitation, run conversation, usage views, network empties; reset affected session/task/loop/automation/network rows (flag, don't retest).
7. Final `make verify` MUST pass once as the program implementation gate before QA tasks.
8. Brand-neutral persisted identifiers MUST remain (`network-participation/v1`); docs may use product vocabulary without rewriting stored atoms.
9. Do not invent spend-cap or mailbox docs — Non-Goals stay Non-Goals.
10. AGH Impact Audit section MUST be completed in completion notes for native tools / extensibility / workspace isolation / official skill.
</requirements>

## Subtasks

- [ ] 6.1 Rewrite network concept pages + guides (`channels-and-peers`, index, coordinate-agents, delivery/safety as needed).
- [ ] 6.2 Rewrite autonomy pages (`coordination-channels.mdx`, `index.mdx`, `coordinator.mdx`) — delete bind-always.
- [ ] 6.3 Update config-toml + lifecycle matrix + protocol pages for broker deletion / live bounds.
- [ ] 6.4 Rewrite official AGH skill references (network, orchestration, native-tools).
- [ ] 6.5 Update glossary + COPY.md mode naming pass.
- [ ] 6.6 Flag `docs/qa/state.csv` rows; run delete-target ripgrep gate; fix stragglers.
- [ ] 6.7 Run `make verify` once; record AGH Impact Audit in completion notes.

## Implementation Details

See TechSpec "Web/Docs Impact", "AGH Impact Audit", "Delete Targets", "No Fallback", Build Order step 9, ADR-003/004/010. Apply `documentation-writer` + `copywriting` + `skill-best-practices` / `writing-great-skills` for context-budget lean skill docs.

Skills to activate: `documentation-writer`, `copywriting`, `skill-best-practices`, `deslop`, `cy-final-verify`.

### Relevant Files

- `packages/site/content/runtime/core/network/*.mdx`
- `packages/site/content/runtime/core/autonomy/coordination-channels.mdx` (+ index/coordinator)
- `packages/site/content/runtime/core/configuration/config-toml.mdx`
- `packages/site/content/runtime/guides/coordinate-agents-over-network.mdx`
- `packages/site/content/protocol/nats.mdx` (retire/rewrite)
- `skills/agh/references/network.md`, `tasks-and-orchestration.md`, `native-tools.md`
- `docs/_memory/glossary.md`, `COPY.md`
- `docs/qa/state.csv`
- TechSpec Delete Targets list as the search corpus

### Dependent Files

- Generated CLI references if not already regenerated in task_05
- Research docs under `.compozy/tasks/network-changes/0*.md` may retain historical mailbox language — exclude from delete gate or mark archival

### Related ADRs

- [ADR-003](adrs/adr-003.md) — discoverability copy
- [ADR-004](adrs/adr-004.md) — one complete release; mailbox removed
- [ADR-010](adrs/adr-010.md) — broker deletion docs impact

## Deliverables

- Site + skill + glossary/COPY hard-cut to Local/Live
- QA tracker flags
- Delete-target gate clean
- `make verify` green
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)** — none assigned; gate is verify + sweep

## Tests

No UT/IT/E2E IDs assigned — this task owns documentation, tracker, and repository sweep gates.

- [ ] Delete-target search gate reports zero production hits for listed symbols/fields/keys/doctrine
- [ ] `make verify` passes once
- [ ] `docs/qa/state.csv` contains new/reset rows for user-visible surfaces from tasks 01–05

## Success Criteria

- Every assigned test case implemented and passing (N/A IDs; gates above green)
- No public doc teaches implicit enrollment or bind-always
- Official skill matches gated tooling + Local/Live semantics
- QA tracker ready for task_07 planning

### Web/Docs Impact

- `web/`: none beyond copy already shipped in task_05 — checked.
- `packages/site`: primary delivery of this task.
- QA impact: tracker edits as specified (this task owns `docs/qa/state.csv` flagging for the program).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: skill/capability docs updated; extension confirmation documented.
- Agent manageability: CLI reference + skill paths document structured verbs and error codes.
- Config lifecycle: config-toml + lifecycle matrix match shipped keys.
