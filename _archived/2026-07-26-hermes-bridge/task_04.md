---
status: pending
title: "Setup CLI: Slack manifest, guided wizards, and verify/doctor/send-test"
type: backend
complexity: high
---

# Task 4: Setup CLI: Slack manifest, guided wizards, and verify/doctor/send-test

## Overview
One-paste Slack manifest generator, guided setup wizards for WhatsApp/Telegram/Discord, and live verify + doctor CategoryBridge + real send-test — the full CLI setup surface. Channel setup today is 8–17 manual dashboard/curl steps with no pre-enable credential probe; this slice collapses that to AGH-driven flows with HTTP/UDS parity so agents and the web orchestrator (task_05) can drive the same paths.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST emit a Slack app manifest containing display info, `oauth_config.scopes.bot` (the enumerated scope set the AGH Slack bridge actually needs — derive from the events the provider ingests: message.*, app_mention, reaction_added, slash commands, chat:write, chat:write for edits, reactions:write for task_03 progress reaction affordances), `settings.event_subscriptions.bot_events`, `interactivity.is_enabled: true`, `socket_mode_enabled: false` (AGH is webhook-based) with the bridge webhook URL templated into request URLs.
- MUST support a bool `--write` with an optional `--out PATH` (avoid Cobra's error-prone optional-flag-value shape) and print exact "paste into api.slack.com → From an app manifest" next steps; default output is structured JSON on stdout (agent-manageable, SD-011).
- MUST expose HTTP **and UDS** parity via the shared `internal/api/core` BaseHandlers (e.g. `GET /api/bridges/providers/slack/manifest?instance=<id>`), co-shipping OpenAPI + TS types — no partial-surface completion (SD-011).
- MUST validate the emitted manifest against Slack's manifest JSON schema in tests, using a VENDORED copy of the schema committed as a test fixture (note source + version; no network in tests — hermetic-test discipline).
- MUST live in a new `internal/cli/bridge_manifest.go` (file-cap discipline; `bridge.go` is already at 12 verbs).
- MUST rewrite the Slack section of `setup.mdx` around the manifest flow (Option A recommended, manual as Option B) and enumerate the scopes.
- MUST provide a setup framework in `internal/cli/bridge_setup.go` (new file) that drives create-instance + secret-binding via the existing client APIs — no new storage paths.
- MUST support non-interactive `--json` input and structured JSON output for every wizard (agents must be able to drive setup — SD-011; Hermes's wizards are TTY-only, a named gap).
- WhatsApp: MUST validate field shapes (phone_number_id 15-17 digit Meta ID vs phone number, access-token prefix, app-secret hex) and detect wrong-product pastes (`sk-`, `xoxb-`, `ghp_` → name the product); MUST auto-generate `verify_token` when absent; MUST print the Meta webhook subscription steps + exact verification curl.
- Telegram: MUST validate token shape (`^\d+:[A-Za-z0-9_-]{30,}$`), generate `webhook_secret`, bind both, and register the webhook via the platform API from the daemon (holds the token post-binding), with `--print-only` to emit the curl instead.
- The daemon-side webhook-registration action MUST be exposed as an HTTP **and UDS** route via the shared `internal/api/core` BaseHandlers (e.g. `POST /api/bridges/:id/webhook/register`), codegen co-shipped — an HTTP-only agent (SD-011) and the web setup orchestrator (task_05) can complete Telegram setup without the CLI (peer review r2 N-003). The CLI wizard calls this same route.
- Discord: MUST build the bot invite URL with the correct scopes/permissions from the instance config and print the Interactions-Endpoint steps.
- MUST mask secrets on echo, keep existing values as defaults on re-run, and end every flow with the exact `agh bridge verify <id>` follow-up.
- MUST NOT depend on any third-party pairing/broker service (rejected C3).
- MUST add `agh bridge verify <id>`: platform identity probe with the bound token (Slack `auth.test` + granted-scope diff vs the manifest scope set + bot-vs-user token detection, Telegram `getMe`, Discord `users/@me`, Teams token acquisition, GChat credential check, WhatsApp token check), webhook-path reachability, and per-check structured results `{check, status, remediation}` — records, not log lines.
- MUST register a doctor `CategoryBridge` running the same probes across instances (`agh doctor --only bridge`).
- MUST add `agh bridge send-test <id> --message …` performing a REAL platform send through the provider (delivery path, not a daemon-side HTTP shortcut); `test-delivery` stays dry-run and its docs say so.
- MUST expose HTTP **and UDS** parity for verify and send-test via the shared `internal/api/core` BaseHandlers (`POST /api/bridges/:id/verify`, `POST /api/bridges/:id/send-test`) with codegen co-ship — no partial-surface completion (SD-011).
- The daemon→provider check request is a bridgesdk CONTRACT ADD: it MUST ship wired across all 8 in-tree adapters in the same change (providers without an identity endpoint return an explicit `skipped`/not-supported check result — never a silent absence), with bridgesdk contract docs and codegen co-shipped (L-025).
- MUST run probes without flipping instance lifecycle state (read-only against the state machine; `report_state` rules unchanged).
- Probes MUST live behind the adapter boundary where they need platform SDKs (daemon asks the provider via a new host-callable or delivery-transport check; do NOT teach the daemon platform REST APIs).
</requirements>

## Subtasks
- [ ] 4.1 Manifest builder reading the installed Slack extension manifest + instance webhook config; scope/event sets enumerated as typed constants (incl. `reactions:write` for task_03 progress reactions)
- [ ] 4.2 CLI verb `agh bridge manifest slack` with `--write`, structured stdout, next-steps epilogue
- [ ] 4.3 HTTP/UDS manifest route + codegen co-ship
- [ ] 4.4 Setup framework: platform descriptor (prompts, validators, secret slots), interactive runner, `--json` runner, structured result
- [ ] 4.5 WhatsApp wizard with the ported validators + wrong-product detection
- [ ] 4.6 Telegram wizard incl. daemon-side `setWebhook` (+ `--print-only`) + HTTP/UDS `POST …/webhook/register` route
- [ ] 4.7 Discord invite-URL builder + interactions-endpoint guidance
- [ ] 4.8 Verify probe contract daemon↔provider + per-platform probe implementations in the providers
- [ ] 4.9 `agh bridge verify` CLI + HTTP route with structured results; doctor `CategoryBridge` registration + probe aggregation
- [ ] 4.10 `agh bridge send-test` CLI + HTTP route through the real delivery path
- [ ] 4.11 `setup.mdx` CLI-first rewrite (manifest/wizard/verify); Web/Docs Impact stubs for task_05 (manifest/verify/send-test/webhook-register shapes)

## Implementation Details
Reference `_techspec.md` §3.4 (setup & verification surface), §3.8 invariants 8–9, §4 (delete targets: curl-first setup instructions in `setup.mdx`), §7–§8 (impact audit + Web/Docs). Keep new CLI code in `bridge_manifest.go`, `bridge_setup.go`, `bridge_verify.go` (file-cap). Scope-diff check consumes the manifest scope constants from this task. Delete targets: curl-first Slack steps 1–7 and curl-first WhatsApp/Telegram/Discord instructions in `setup.mdx` (replaced by CLI-first; manual path remains as Option B / fallback appendix). Cross-refs updated: progress reactions → task_03; web hand-off → task_05.

### Relevant Files
- `internal/cli/bridge_manifest.go` — Slack manifest builder + CLI verb
- `internal/cli/bridge_setup.go` — setup framework + WhatsApp/Telegram/Discord wizards
- `internal/cli/bridge_verify.go` — verify + send-test CLI verbs
- `internal/doctor/` — CategoryBridge + probe aggregation
- `internal/api/core/bridges.go` — manifest/verify/send-test/webhook-register routes
- `extensions/bridges/*/provider.go` — per-platform probe handlers
- `packages/site/content/runtime/core/bridges/setup.mdx` — CLI-first rewrite

### Dependent Files
- `internal/cli/bridge_test.go` — manifest/wizard/verify suites
- `internal/cli/cli_integration_test.go` — daemon round-trips
- `internal/doctor/doctor_test.go` — CategoryBridge cases
- `internal/extension/` — check-request transport plumbing
- `openapi/` + `web/src/generated/` — codegen co-ship for new routes
- `packages/site/content/runtime/cli-reference/bridge/` — regenerated verb pages

### Competitor References
- `.resources/hermes/hermes_cli/slack_cli.py:26-200` — manifest builder + `--write` + next steps
- `.resources/hermes/tests/hermes_cli/test_slack_cli.py:48,84,97` — scope↔event relation tests
- `.resources/hermes/hermes_cli/setup_whatsapp_cloud.py:52-157,273-503` — validators + flow
- `.resources/hermes/hermes_cli/telegram_managed_bot.py:35` — token regex
- `.resources/hermes/plugins/platforms/slack/adapter.py:845-909` — scope-diff + bot-token diagnostics
- `.resources/hermes/hermes_cli/doctor.py:511` — sectioned check/warn/fail model

## Deliverables
- `agh bridge manifest slack` + HTTP/UDS parity + rewritten Slack setup docs
- `agh bridge setup {whatsapp,telegram,discord}` interactive + `--json`
- Daemon-registered Telegram webhook; Discord invite URL; WhatsApp validated flow
- `agh bridge verify`, doctor `CategoryBridge`, `agh bridge send-test` + HTTP/UDS parity
- Web/Docs Impact stubs documenting shapes consumed by task_05
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests
Cases assigned from `_tests.md` (old task_08 + task_09 + task_10 ownership). Read each case's full definition there before writing tests.

### Unit — Slack manifest (`internal/cli/bridge_test.go`)
- [ ] Should emit a manifest that validates against the VENDORED Slack app-manifest JSON schema fixture (no network; fixture header notes source + version)
- [ ] Should hold the scope↔event RELATIONS: every subscribed `message.*`/`app_mention`/`reaction_added` event has its required read scope; `chat:write` always present; reaction events ↔ `reactions:write` (invariant, not snapshot — analysis 04 §3.5)
- [ ] Should template request URLs from the instance webhook path and set `socket_mode_enabled: false` (AGH is webhook-based)
- [ ] Should write the file under `--write` (with `--out PATH` override) and print the exact "paste into api.slack.com → From an app manifest" epilogue; a bare invocation emits valid JSON on stdout only (agent-manageable, SD-011)

### Unit — Setup wizards (`internal/cli/bridge_test.go`)
- [ ] Should reject a phone number pasted into `phone_number_id` with the "look below the From dropdown" remediation, name the product on wrong-product pastes (`sk-…` → OpenAI key, `xoxb-…`, `ghp_…`), and pass valid WhatsApp field shapes (validators ported from `setup_whatsapp_cloud.py:52-157`)
- [ ] Should accept a valid BotFather token against `^\d+:[A-Za-z0-9_-]{30,}$` and reject truncated ones
- [ ] Should auto-generate `verify_token`/`webhook_secret` when absent and keep existing values as masked defaults on re-run (inv9)
- [ ] Should complete `--json` mode with zero prompts: full config in → instance created + secrets bound + structured result out (SD-011)
- [ ] Should build the Discord invite URL with the required scopes and permission bits from the instance config

### Unit — Verify / send-test / doctor (`internal/cli/bridge_test.go`, `internal/doctor/doctor_test.go`, `extensions/bridges/slack/provider_test.go`)
- [ ] Should run bridge probes only under `agh doctor --only bridge`; a failing identity probe yields status `fail` with a non-empty remediation naming the secret slot
- [ ] Should render `verify` as structured JSON with per-check `{check, status, remediation}` records and exit non-zero on any fail (records, not log lines — analysis 02 §5)
- [ ] Should route `send-test` through the REAL delivery path (fake transport records a platform send) while `test-delivery` still performs zero platform calls (D2)
- [ ] Should report a fake `auth.test` missing `mpim:history` as a check result naming the missing scope and the reinstall requirement, and a user token (no `bot_id`) as "user token, expected bot token" (scope diff vs the manifest scope set)

### Integration (`make test-integration`)
- [ ] Should round-trip `agh bridge manifest slack` against a daemon with the Slack extension installed and return the manifest for a real instance id (suite: `cli_integration_test.go`)
- [ ] Should complete the wizard `--json` round-trip against a daemon: instance exists, bindings listed, Telegram `setWebhook` hit the fake platform endpoint — or was printed under `--print-only`
- [ ] Should register the Telegram webhook via `POST /api/bridges/:id/webhook/register` with ZERO CLI involvement — HTTP-only agent parity proof, status + body asserted (SD-011; peer review r2 N-003)
- [ ] Should aggregate verify checks through the daemon→provider→daemon round-trip WITHOUT changing instance lifecycle state (inv8)
- [ ] Should get an answer to the check request from ALL 8 in-tree providers: identity-capable ones return real probe results, the rest return explicit `skipped` — none silently absent (inv7, L-025 hard-cut)
- [ ] Should co-ship OpenAPI: every new bridge route (`GET …/providers/slack/manifest`, `POST …/:id/verify`, `POST …/:id/send-test`, `POST …/:id/webhook/register`) is present in the generated contract + generated TS types, and each endpoint asserts status AND body (`eng-contract-codegen-coship`)

### E2E — Runtime (`make test-e2e-runtime`, bridge contract lane)
- [ ] Should cover the verify and send-test request/response shapes in the contract mocks

Test coverage target: >=80% for touched packages. All tests must pass under the repo gates (`-race` for Go).

## Success Criteria
- Every assigned test case implemented and passing
- Slack setup is paste-manifest → install → bind (docs path drops from 8 dashboard steps)
- Telegram zero-to-bound without user-run curl; WhatsApp misconfigurations caught before any Meta API call
- A bridge with an invalid token fails `agh bridge verify` BEFORE enable with an actionable remediation; a valid one delivers a real test message via `send-test`
- The scope list exists exactly once in code and docs render from/match it (drift test in later conformance task references it)
