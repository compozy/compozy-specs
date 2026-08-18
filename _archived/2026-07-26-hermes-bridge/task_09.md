---
status: pending
title: Plan the bridges QA cycle (qa-report)
type: test
complexity: medium
---

# Task 9: Plan the bridges QA cycle (qa-report)

## Overview
Plans the QA cycle covering progress, setup, and delivery robustness — journeys, `state.csv`
rows, persona charters — for the resliced program (tasks 01–08). This program changes
user-visible behavior across CLI verbs, bridge delivery UX, web UI, and docs; per SD-005 the
qa-report + qa-execution pair is the gate that catches what `make verify` cannot. Start from
`_qa.md`; do not re-derive it.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST activate the `qa-report` skill with `qa-docs-path=docs/qa` — it owns the living tree
  contract (`docs/qa/{state.csv,journeys/,charters/,bugs/,reports/,automation-backlog.md}`).
- MUST ALWAYS READ `_qa.md` (personas, J-A..J-G journey drafts, NB-047+ row seeds, CH-a..h
  charter seeds, risk taxonomies, TTFM protocol, §10 completeness gate) before starting —
  start from it, do not re-derive it.
- MUST map the new/changed journeys as Mermaid flows BEFORE deriving scenarios: (1) zero-to-
  first-message per channel via manifest/wizard; (2) watch-the-agent-work in a channel
  (progress modes, threading, completion lines); (3) misconfiguration → verify → remediation;
  (4) long-reply chunking + markdown fidelity; (5) restart-mid-turn recovery; (6) web setup
  orchestrator; (7) inbound edits & reply context — covering the resliced program tasks 01–08
  (not the old 01–16 numbering).
- MUST add `docs/qa/state.csv` rows: `untested` for every new behavior (manifest, setup
  wizards, verify, send-test, doctor bridge, progress rendering per platform, web checklist,
  inbound edit/reply, durable ledger recovery); reset affected existing bridge rows
  (NB-025..NB-046 class per `_qa.md` §4) to `untested` where behavior changed — flag, don't
  retest.
- MUST derive persona-driven charters (operator setting up their first Slack bridge; agent
  driving setup via `--json`/HTTP per SD-011; teammate watching a channel mid-run; bridge
  operator robustness + security lens).
- MUST register expected-bug taxonomies for the risk areas the analyses flagged: progress spam/
  rate limits (analysis 07 G10/G11), markdown escaping (G12), restart recovery, redaction
  bypass, thread misrouting, UTF-16 boundaries, wizard false-accepts, checklist truthfulness,
  transcript purity.
- MUST keep every journey walked by a persona (qa-report completeness rule).
- MUST embed the time-to-first-message measurement protocol (Hermes baselines: Slack ≈ 7
  actions, Telegram ≈ 4) into the relevant charter missions.
- MUST hand off a concrete plan for task_10 (qa-execution) — not for the old task_18 id.
</requirements>

## Subtasks
- [ ] 9.1 Journey flowcharts for J-A..J-G (from `_qa.md` §3) written as
      `docs/qa/journeys/J-NN-*.md` with Mermaid flows, true end states, and ≥1 abandonment path
      each; map onto `_tests.md` §6/§7 E2E lanes
- [ ] 9.2 `state.csv` rows added/reset: mint NB-047+ seeds from `_qa.md` §4; reset changed
      NB-024..NB-039-class rows per the final program diff; `qa_status=untested` (flag, don't
      retest)
- [ ] 9.3 Session charters per persona (CH-a..h seeds from `_qa.md` §5) with concrete
      inputs/expected outcomes — zero "test the happy path" charters; include the agent-driven
      structured-output lane (SD-011)
- [ ] 9.4 Time-to-first-message measurement protocol embedded in CH-a/CH-b/CH-c missions
      (step counts per channel, before/after vs Hermes baselines)
- [ ] 9.5 Completeness gate (`_qa.md` §10): every behavior→charter, every journey→persona;
      risk taxonomies reflected as charter hunt targets; automation-backlog seeds recorded

## Implementation Details
Skill-owned formats: follow `qa-report` for tree layout, charter shape, BUG-NNNN registry
conventions, and state-schema columns. No production code changes. Scope is the resliced
program (tasks 01–08); execution is task_10. Reference `_tests.md` §6/§7 as the automated
backbone journeys map onto. Area code stays `NB` — no new area minted.

### Relevant Files
- `_qa.md` — QA base (personas, J-A..J-G, NB-047+ seeds, CH-a..h, §6 taxonomies, §7 TTFM,
  §10 completeness gate = this task's exit criteria)
- `_tests.md` — automated backbone the journeys map onto
- `_techspec.md` — user-visible surfaces and invariants under test
- `docs/qa/state.csv`, `docs/qa/journeys/`, `docs/qa/charters/`, `docs/qa/personas.md`,
  `docs/qa/automation-backlog.md`

### Dependent Files
- `task_10.md` — executes this plan
- `docs/qa/README.md` — untouched except optional changelog; reuse `NB` area

### Competitor References
- `.resources/hermes/website/docs/user-guide/messaging/index.md:18-46` — capability matrix as
  a coverage checklist for per-channel scenario derivation

## Deliverables
- Updated `docs/qa/` tree: journeys (J-A..J-G), charters (persona-driven), `state.csv` rows
  added/reset, automation backlog seeds
- Completeness gate recorded and green
- Ready handoff for task_10 (`eng-qa-bootstrap` lab execution)

## Tests

This task's deliverable IS the test plan — no production code. Completeness gate from
`_qa.md` §10 / `qa-report` Step 7:

- Unit tests (suite: Not applicable — QA planning artifact; no code):
- Integration tests (suite: Not applicable — execution happens in task_10):
- E2E tests (lane: Not applicable — this task plans the manual+scripted cycle):
- Completeness gate (REQUIRED — treat as the Tests checklist):
  - [ ] Every journey (J-A..J-G) exists as a `docs/qa/journeys/` file with a Mermaid flow and
        an abandonment path, and is walked by ≥1 assigned persona charter
  - [ ] Every seed row from `_qa.md` §4 minted (or consciously merged/dropped with a note)
        with `journey` pointing at a real journey file; resets from §4 applied per the final
        diff of tasks 01–08
  - [ ] Every new/changed behavior row maps to a charter; every charter names concrete
        inputs/expected outcomes (e.g. "6000-char reply on Discord fake → 3 ordered messages")
  - [ ] Risk taxonomies (`_qa.md` §6) reflected as charter hunt targets, not just prose
  - [ ] TTFM protocol (`_qa.md` §7) embedded in CH-a/CH-b/CH-c missions
  - [ ] `docs/qa/README.md` untouched except (if needed) changelog; `NB` area reused — no new
        area code minted
  - [ ] Coverage scope is tasks 01–08 (resliced), not the archived 01–16 numbering; handoff
        names task_10 (not old task_18)

## Success Criteria
- Every completeness-gate item above checked
- `docs/qa/state.csv` reflects every user-visible change from tasks 01–08
- Charters name concrete inputs/expected outcomes — zero "test the happy path" charters
- Plan is executable by task_10 against a fresh `eng-qa-bootstrap` lab
