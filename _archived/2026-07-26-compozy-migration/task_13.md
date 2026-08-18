---
status: completed
title: Real-User QA Execution
type: qa-execution
complexity: critical
---

# Task 13: Real-User QA Execution

## Overview

Executes the planned QA cycle against a real local runtime and produces fresh pre-publish evidence for the single v0.3 beta cut. This is the last local/release-PR gate before Pedro may perform any external release action: the `legacy/v0.2` branch, the squash merge of `v0.3` into `main`, the beta publish, the `compozy.com` pointing, and the `compozy/agh` archival all wait on this task being green (Task 10's single-cut runbook). Live installer, registry, and cosign acceptance are deliberately post-publish checks in that runbook.

<critical>
- ALWAYS READ the exact scenario IDs named by this cycle's charters, the open `docs/qa/bugs/` registry, and those charters in `docs/qa/charters/` before executing
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — do not simulate a published beta, replace browser evidence with shell output, weaken a regression test, or bypass a failing gate with a manual verdict
</critical>

<requirements>
- MUST activate `qa-execution` with the repo QA docs path, and `eng-real-scenario-qa` for release-grade runtime scope (playbook lab, in-persona kickoff, runtime observation, strict audit).
- MUST execute exactly the deduplicated scenario-ID union named by Task 12's charters: the invariant-derived targeted tier plus one adjacent canary. Do not expand scope to every preexisting `qa_status: untested` record; tasks 01-11 coverage is proven by scenario `entry_points`, while unselected backlog remains for later cycles.
- MUST bootstrap the lab through `eng-qa-bootstrap` and consume the manifest's unique runtime home, daemon ports, and bridge socket. Activate `eng-worktree-isolation` only when concurrency is explicitly signaled; default home and port remain forbidden for any concurrent run.
- MUST export the manifest-derived `COMPOZY_WEB_API_PROXY_TARGET` before starting Web/browser QA and use that value for `make test-e2e-web`; never hardcode the daemon proxy target.
- MUST run `make test-e2e-runtime` (daemon harness) AND `make test-e2e-web` (Playwright), driving the highest-risk UI workflow through the browser skill with the documented fallback; shell-only substitutes are not acceptable evidence.
- MUST exercise the CLI, HTTP, and UDS surfaces for the runtime-selection and input-default paths, comparing structured CLI output against HTTP/UDS responses for the same persisted state.
- MUST NOT require the deferred config migrator or first-boot legacy-state journeys in this cycle — those belong to task 14. Verify instead that both migration-guide surfaces document the migrator as deferred (task 14) and that active upgrade guidance (command map, input defaults, deep link) matches the shipped runtime.
- MUST collect fresh IT-017 release-PR evidence from the candidate ref using the workflow and vendored-skill pins for `github.com/compozy/releasepr@v0.0.24`. It proves `pr-release plan` resolves the explicit ref to checked-out `HEAD`, rejects leading-`v` input and local/`origin` tag collisions, emits the complete authoritative beta output set, and is consumed without downstream ref/version/tag/channel or GitHub/npm/Homebrew policy re-derivation. It also proves the workflow retains annotated tag creation and installer contract fields; it MUST NOT publish or use registry/cosign acceptance as a substitute for post-publish validation.
- MUST verify both migration-guide surfaces carry the same normalized content with the reproducible parity command recorded below, and that the disposition ledger accounts for every audited legacy CLI/web/extension/SDK surface with a successor or removed/deferred status and workaround.
- MUST verify the launch surfaces shipped in-branch by Task 11: the landing renders the locked text with an OS-shell capture, the site's install/version copy matches the beta channel truth and the front-door README, the `compozy.com` canonical origin holds, and no UI renders a control the runtime does not support. Rendering verification is local; no hosting route or DNS is touched.
- MUST register every reproduced defect in the content-addressed bug registry after deduplicating, and link it from the affected scenario files.
- MUST follow the fix-loop governor: before changing a test, name its invariant, owning layer, and canonical suite; make only a small contained production fix with red-before/green-after evidence in that suite, one logical fix per commit. Do not add static prose/CSS/snapshot tests unless the artifact is the product contract and no stronger gate owns it. Everything else escalates to a decisions-for-a-human section.
- MUST update scenario verdicts and write the dated run report; re-run `make verify`, then invoke the manifest's `AUDIT_COMMAND` with `--qa-output-path "$QA_OUTPUT_PATH" --final-report "docs/qa/reports/<YYYY-MM-DD>-<playbook-ref>.md" --strict`. Auditor exit code 2 or any blocking check prevents a green verdict; cite `qa-audit-report.json` as evidence.
- <critical>MUST tear down every lab process on every terminal path — pass, fail, blocked, or abort — via the manifest teardown command or the reaper, and MUST cite the teardown evidence showing a clean state. Leaving lab daemons, tmux servers, dev servers, browsers, or watchers alive is a blocking failure.</critical>
</requirements>

## Subtasks

- [x] 13.1 Bootstrap an isolated lab with unique runtime home, ports, and sockets; activate worktree isolation only when concurrency is signaled; register long-lived process pids
- [x] 13.2 Resolve and record the exact charter scenario-ID union, then execute only those targeted/canary sessions in persona
- [x] 13.3 Export the manifest-derived Web proxy target, run the daemon E2E harness and browser E2E lane, and drive the highest-risk UI workflow through the browser skill
- [x] 13.4 Exercise CLI/HTTP/UDS parity for runtime selection and input defaults against the same persisted state
- [x] 13.5 Confirm the migration guide marks the config migrator/first-boot probe as deferred (task 14) and that active upgrade guidance matches the shipped runtime
- [x] 13.6 Capture fresh IT-017 evidence for the pinned `releasepr v0.0.24` planner, its fail-closed ref/tag guards, authoritative outputs, and workflow-owned annotated tag creation; hand live installer, registry, and cosign acceptance to Task 10's post-publish checklist
- [x] 13.7 Verify both guide surfaces' section parity and the completeness of the deferred section
- [x] 13.8 Verify the landing hero and confirm no unsupported controls render anywhere
- [x] 13.9 Register deduplicated bugs, link them from scenarios, and apply only governor-compliant fixes
- [ ] 13.10 Update scenario verdicts, write the dated run report, re-run `make verify`, then run and cite the strict manifest evidence audit with no blocking result
- [x] 13.11 Tear down the lab and cite clean teardown evidence

Operator closeout on 2026-07-27: the report and scenario verdicts are current, but the final
post-freeze `make verify` and strict evidence audit in 13.10 were explicitly waived to meet an
urgent external application deadline. The report preserves that gap and the open RT-073 P1; no
green-gate claim is made for the waived checks.

## Implementation Details

Evidence lives in the living QA tree: scenario verdicts, the content-addressed bug registry, and a dated run report. The lab holds only run-scratch evidence indexed by path.

Provider-home policy follows each provider's contract: bound-secret and brokered lanes isolate the provider home, while native-CLI providers with operator home policy preserve the operator login unless a scenario explicitly tests isolation.

Highest-risk observation targets: per-task runtime truthfulness (status values must trace to what the sessions actually received), review artifact byte-shape and round atomicity, truthful deferred-migrator documentation, and release-PR evidence integrity. Release evidence must identify the pinned `releasepr v0.0.24` module, the exact candidate commit, the planner outputs, and the workflow step that consumes them and creates the annotated tag. Live migrator non-destruction waits on deferred task 14. The live beta install/registry boundary is observed only after an actual publish through Task 10's single-cut runbook.

Run the canonical guide-parity target from the repository root after Task 09 has created both guides. The target invokes `scripts/verify-migration-guide-parity.sh`, normalizes frontmatter and site-only MDX, compares all eight required section IDs and canonical content, prints a diff, and exits non-zero on drift:

```sh
make migration-guide-check
```

### Relevant Files

- `docs/qa/charters/` — this cycle's charters from task 12
- `docs/qa/scenarios/` — only the exact scenario IDs listed by Task 12's cycle charters; other `untested` records remain backlog, although every tasks 01-11 surface must appear in some scenario `entry_points`
- `docs/qa/bugs/` — content-addressed registry to dedupe against and extend
- `docs/qa/reports/` — the dated run report for this cycle
- `internal/testutil/e2e/**` and `internal/e2elane/lanes.go` — daemon E2E harness and lane definitions
- `web/e2e/` — browser E2E suite
- `MIGRATION_GUIDE.md` and `packages/site/content/runtime/migration/` — the two guide surfaces to compare

### Dependent Files

- `docs/qa/state.csv` — regenerated view; never hand-edited
- Task 10's runbooks — unblocked only when this task is green

### Related ADRs

- All eight ADRs are in observation scope; ADR-005 (distribution identity, license framing) carries the highest user-visible risk in this cycle. ADR-006 (non-destructive migrator) is documented as deferred via task 14 and is not an executable gate here. ADR-008 binds the provider-free review journey and excludes the retired CodeRabbit/watch path.

## Authorized Scenario Scope

Task 13 executes these exact 25 deduplicated IDs, as selected by Task 12's eight targeted charters
and one adjacent-canary charter:

1. `RT-compozy-cli-binary`
2. `RT-compozy-global-database`
3. `RT-compozy-home-layout`
4. `RT-compozy-home-isolation`
5. `RT-compozy-environment-namespace`
6. `ET-compozy-native-tool-invocation`
7. `ET-compozy-extension-contract-identity`
8. `ET-compozy-official-skill-discovery`
9. `NB-compozy-wire-identity`
10. `RT-compozy-claim-token-redaction`
11. `ET-compozy-public-brand-navigation`
12. `LP-runtime-selection-overrides`
13. `LP-runtime-provenance-observation`
14. `LP-loop-run-deep-link`
15. `LP-loop-input-defaults`
16. `LP-runtime-validation-preflight`
17. `LP-agent-authored-review-run`
18. `LP-review-artifact-inspection`
19. `LP-review-round-finalization`
20. `ET-dev-cycle-skill-bundle`
21. `ET-dev-cycle-legacy-skill-retired`
22. `REL-release-candidate-plan`
23. `REL-migration-guide-parity`
24. `REL-beta-channel-contract`
25. `REL-os-landing-proof` — the only adjacent canary.

Do not select `REL-beta-install-paths`, `REL-beta-installer-provenance`, or
`REL-beta-self-update`; they require a real publication and remain Task-10 post-publish backlog.
Do not select the Task-14 live migrator/first-boot journey or historical provider-backed review
rows such as `LP-029`.

## Deliverables

- Executed charter sessions with runtime observation evidence
- Green daemon and browser E2E lanes with the highest-risk UI workflow driven through the browser skill
- Verified deferred-migrator guide wording, fresh IT-017 release-PR evidence, guide parity, and local hero rendering
- Deduplicated bug registrations linked from affected scenarios
- Updated scenario verdicts, a dated run report, a cited strict `qa-audit-report.json` with no blocking result, and cited clean teardown evidence
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from [`_tests.md`](_tests.md), the test contract — read each ID's full definition there before writing tests.

- [ ] No `_tests.md` IDs are assigned to this task — it executes the QA cycle rather than authoring new automated cases. It DOES run the full automated contract as a gate: `make verify`, `make test-integration`, `make test-e2e-runtime`, and `make test-e2e-web` with the manifest-derived proxy target, plus fresh IT-017 release-PR dry-run evidence and the manifest's strict evidence audit. Post-publish checks are not faked in this lane.

### Web/Docs Impact

- `web/`: exercised, not modified — `web/e2e/**` runs; any web fix must follow the fix-loop governor with a regression test. Checked surfaces: loop run views (per-task runtime display), OS shell, session views.
- `packages/site`: exercised, not modified — both guide surfaces, the landing hero, install instructions, and generated references are verified for truthfulness against the shipped runtime.
- QA impact: this task IS the QA cycle — it updates scenario verdicts and registers bugs. Verdict updates are the expected output, not a flag violation.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: exercised — bundled skills projecting into a managed session, the dev-cycle extension's new tools, renamed native tool IDs, and the renamed official runtime skill are all verified through their real surfaces.
- Agent manageability: exercised — structured CLI output compared against HTTP and UDS for the same state, and deterministic error contracts triggered deliberately (unknown provider/model, unknown input key). Second migrator-run and legacy-state warning checks wait on deferred task 14.
- Config lifecycle: exercised — the new runtime and input-default keys are set through real config files at global and workspace scope, with merge/override behavior and validation errors observed end to end. Live legacy-config translation waits on deferred task 14. Config writes against a single isolated home run sequentially, never in parallel.

### Compozy Impact Audit

- Native tools: exercised, not changed — verify renamed tool IDs, descriptors, schema digests, capability gates, and CLI/API fallbacks against the candidate runtime; do not alter tool contracts in this QA task.
- Extensibility and hooks: exercised, not changed — verify extensions, hooks, skills/capabilities, tools/resources, bundles, registries, bridge SDKs, MCP sidecars, and config lifecycle through the planned scenarios.
- Workspace data isolation: exercised, not changed — verify global/workspace/session/agent scope and workspace propagation through CLI/HTTP/UDS/core/store/web/SSE/cache/events where the selected scenarios touch it; no QA evidence may use the default shared home.
- Official Compozy skill: exercised, not changed — verify `skills/compozy/` where the planned public behavior requires it; no public tool, CLI path, hook event, capability, bundle/resource, or memory/network/task semantic changes are authored here.

## Success Criteria

- Every assigned test case implemented and passing
- All gates green: `make verify`, integration, daemon E2E, and browser E2E
- Every charter-listed targeted/canary scenario carries an updated verdict, no unlisted backlog scenario is pulled in implicitly, and every reproduced defect is registered and linked
- Fresh IT-017 release-PR evidence records the pinned `releasepr v0.0.24` planner, candidate-ref/HEAD equality, local/remote tag guards, complete authoritative outputs, and workflow-owned tag/publication boundary; the beta live boundary remains explicitly deferred to Task 10's post-publish checklist, and live migrator/first-boot journeys remain deferred to task 14
- Both guide surfaces match in normalized content, mark the config migrator as deferred, and the legacy-surface disposition ledger is complete
- A dated run report is written, the strict evidence audit has no blocking result, and clean teardown evidence is cited
