---
status: pending
title: Progress rendering across six channel providers
type: backend
complexity: high
---

# Task 3: Progress rendering across six channel providers

## Overview
Renders live tool progress via the accumulator on Slack/Telegram/Discord (default on) and Teams/GChat/WhatsApp (default off, opt-in); GitHub/Linear ignore Progress cleanly. With the progress contract, labels, and accumulator from task_01 in place, this is where the user finally sees "the bot posts each tool call as it works" — edit-capable providers get threaded bubbles; post-oriented providers get conservative sparse status posts.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST render progress events via the bridgesdk accumulator: Slack `chat.postMessage` + `chat.update`, Telegram `sendMessage` + `editMessageText`, Discord message create + edit.
- MUST thread progress AND the final answer under the user's triggering message on Slack (thread_ts from the inbound event) — channel stays readable with concurrent users.
- MUST show typing: Slack `assistant.threads.setStatus`, Telegram `sendChatAction`, Discord typing endpoint — cleared when the final answer posts.
- MUST add processing reactions where supported: 👀 on turn start, ✅/❌ on completion.
- MUST render completion lines for long tools (✅ verb + duration / ❌ verb + short error).
- MUST honor the resolved per-instance mode (`off` produces zero platform calls).
- MUST implement Slack `chat.delete` so a future `cleanup_progress` option is possible (Hermes F5 gap); cleanup itself stays default-off.
- MUST respect platform rate limits: accumulator throttle + `RetryAfter` honored on edits.
- MUST render progress on Teams (accumulator via activity update, `TextFormat: markdown`) and GChat (accumulator via message patch) when the instance opts in.
- MUST render WhatsApp progress as sparse one-line status posts (grouping `separate`, mode `new` semantics) — no edit-in-place.
- MUST default Teams/GChat/WhatsApp to `off`; enabling is a per-instance `delivery_defaults.progress` change only (no code).
- MUST keep GitHub/Linear providers progress-free but compiling cleanly against the new contract (they ignore `Progress` payloads — they are issue bridges, not chat surfaces).
- MUST honor typing affordances where they exist (Teams typing activity); no-op elsewhere.
</requirements>

## Subtasks
- [ ] 3.1 Slack: progress bubble via accumulator, thread_ts routing, setStatus typing, reactions, `chat.delete` (new `progress.go` — do not append to oversized `provider.go`)
- [ ] 3.2 Telegram: status bubble edits with MarkdownV2-safe text, sendChatAction, reactions
- [ ] 3.3 Discord: message create/edit rendering, typing, reactions
- [ ] 3.4 Teams: progress rendering (accumulator + typing activity) in a new per-provider `progress.go`
- [ ] 3.5 GChat: progress rendering (accumulator via message patch)
- [ ] 3.6 WhatsApp: sparse status-post rendering (`separate` grouping)
- [ ] 3.7 GitHub/Linear: compile-and-ignore verification against the Progress contract
- [ ] 3.8 Per-provider rendered-payload contract tests asserting exact platform API bodies (Hermes F9 fix)
- [ ] 3.9 Provider READMEs note progress behavior + default-on/off per platform

## Implementation Details
Depends on task_01 (progress contract, labels, ProgressAccumulator). Reference `_techspec.md` §3.1/§3.3 and Open Decision #1. Keep provider files under control: rendering goes in a new `progress.go` per provider directory, not appended to `provider.go` (file-cap rule — slack provider.go is already oversized). No GChat CardV2 typing card in v1 (deferred with outbound media); plain text status lines instead. Dialect formatters from task_02 SHOULD be applied on progress lines when that task has landed; if executing in parallel, wire the call sites and assert once both are green.

### Relevant Files
- `extensions/bridges/slack/{provider.go,progress.go}` — Slack progress rendering
- `extensions/bridges/telegram/{provider.go,progress.go}` — Telegram progress rendering
- `extensions/bridges/discord/{provider.go,progress.go}` — Discord progress rendering
- `extensions/bridges/teams/{provider.go,progress.go}` — Teams opt-in progress
- `extensions/bridges/gchat/{provider.go,progress.go}` — GChat opt-in progress
- `extensions/bridges/whatsapp/{provider.go,progress.go}` — WhatsApp sparse status posts

### Dependent Files
- `extensions/bridges/{slack,telegram,discord,teams,gchat,whatsapp,github,linear}/provider_test.go` — rendered-payload assertions
- `extensions/bridges/*/provider_delivery_test.go` — full-turn fake-transport cases
- `extensions/bridges/*/README.md` — progress behavior + default notes

### Competitor References
- `.resources/hermes/plugins/platforms/slack/adapter.py:1352-1544,2048-2061` — post/edit/setStatus/reactions
- `.resources/hermes/gateway/run.py:482-491,16775-16801` — progress thread resolution + fenced terminal commands
- `.resources/hermes/plugins/platforms/telegram/adapter.py:3786` — status bubble
- `.resources/hermes/gateway/display_config.py:92-160` — low-tier defaults rationale
- `.resources/hermes/plugins/platforms/teams/adapter.py:1185` — typing activity

## Deliverables
- Live progress rendering on Slack/Telegram/Discord (default on), mode-gated, threaded, throttled
- Opt-in progress rendering on Teams/GChat/WhatsApp; GitHub/Linear contract-clean
- Rendered-payload contract tests asserting exact API bodies per provider
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests
Cases assigned from `_tests.md` (old task_05 + task_06 ownership). Read each case's full definition there before writing tests.

### Unit — Progress rendering Slack/Telegram/Discord (`extensions/bridges/<provider>/provider_test.go`, per provider)
- [ ] Should render `ToolProgress{phase: started}` as a platform post whose body carries the labeled line and the inbound `thread_ts` (Slack `chat.postMessage`; Telegram `sendMessage`; Discord create) — thread routing keeps channels readable
- [ ] Should render a second progress event inside the window as an edit with both lines against the SAME message id (`chat.update` / `editMessageText` / Discord edit)
- [ ] Should render phase `failed` as a ❌ line with a short error and a ≥30s `completed` as ✅ + duration (completion lines — Hermes F3 divergence)
- [ ] Should make zero platform API calls for progress events on a mode-`off` instance
- [ ] Should start typing on the first progress event (Slack `assistant.threads.setStatus`, Telegram `sendChatAction`, Discord typing) and clear it when the final answer posts
- [ ] Should honor a 429 `Retry-After` on edits with backoff and no dropped event (rate-limit discipline, G10/G11)
- [ ] Should execute Slack `chat.delete` against the fake API (capability for a future `cleanup_progress`; default-off — Hermes F5)

### Unit — Progress rendering Teams/GChat/WhatsApp + issue bridges (per provider)
- [ ] Should make zero progress platform calls on the default (off-tier) instance config
- [ ] Should render progress on an opted-in Teams instance as activity create then update with `TextFormat: markdown`
- [ ] Should render progress on an opted-in GChat instance as message create then patch
- [ ] Should render two DISTINCT tool ids on an opted-in WhatsApp instance as two separate status posts and collapse identical consecutive ids into one (`separate` grouping)
- [ ] Should ack a delivery request carrying a `Progress` payload on GitHub/Linear without any platform side effect (issue bridges stay progress-free but contract-clean; inv7)

### Integration (`make test-integration`; suite: `extensions/bridges/*/provider_delivery_test.go`)
- [ ] Should render a full turn `start → progress×3 → final` over the fake transport: bubble posted, edited twice, final answer posted separately in the same thread (Slack/Telegram/Discord)
- [ ] Should render an opted-in low-tier (Teams/GChat/WhatsApp) turn as ordered status messages over the fake transport and still deliver the final answer

### E2E — Runtime (`make test-e2e-runtime`, bridge contract lane)
- [ ] Should ack progress events in order through the bridge lane for a tool-heavy turn
- [ ] Should round-trip an opted-in low-tier provider through the updated contract mocks

Test coverage target: >=80% for touched packages. All tests must pass under the repo gates (`-race` for Go).

## Success Criteria
- Every assigned test case implemented and passing
- Rendered-payload tests assert exact platform API bodies (mrkdwn text, thread_ts, edit ts) per provider — the invariant Hermes never tested (F9)
- Flipping `delivery_defaults.progress.tool_progress` to `all` on a Teams fake instance turns on rendering with no code change
