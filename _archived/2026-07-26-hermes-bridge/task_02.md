---
status: pending
title: "Outbound text pipeline: shared chunking and platform markdown dialects"
type: backend
complexity: high
---

# Task 2: Outbound text pipeline: shared chunking and platform markdown dialects

## Overview
Shared ChunkMessage in bridgesdk wired to every provider, plus Slack mrkdwn and Telegram MarkdownV2 formatters on all outbound paths (text, chunks, progress lines). Long agent replies hard-fail on most channels today (only WhatsApp chunks), and raw markdown mangles or rejects on Slack/Telegram — this slice makes outbound text delivery robust and dialect-correct across the six chat providers.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST provide `ChunkMessage(text string, limit int, lenFn func(string) int) []string` in `internal/bridgesdk` that preserves fenced code blocks across chunks (close fence at chunk end, reopen with the same language tag) and appends `(N/M)` indicators for multi-chunk output.
- MUST accept a custom length function; Telegram uses UTF-16 code-unit length.
- MUST declare a per-provider max message length: Slack 40000, Telegram 4096 (UTF-16), Discord 2000, Teams 28000, GChat per its API cap, WhatsApp 4096.
- MUST loop chunks on `post` operations; `edit` operates on the last chunk and posts continuations for the rest.
- MUST delete the WhatsApp-local `splitMessage` in favor of the shared helper (zero-legacy).
- MUST NOT change the daemon↔adapter contract — chunking is provider-side.
- MUST convert standard markdown → Slack mrkdwn: `**bold**`→`*bold*`, `~~strike~~`→`~strike~`, `[text](url)`→`<url|text>`, escape `<>&`, protect code spans/fences from conversion.
- MUST escape Telegram MarkdownV2 specials (`_*[]()~`>#+-=|{}.!`) outside code spans and send with `parse_mode: MarkdownV2`; fall back to plain text when escaping would corrupt content.
- MUST apply the formatter on every outbound path: text deliveries, chunk continuations, and progress lines.
- MUST keep conversion pure and table-testable (input → expected output corpus).
</requirements>

## Subtasks
- [ ] 2.1 `ChunkMessage` helper in `internal/bridgesdk/chunk.go` with fence repair, `(N/M)` indicators, and pluggable length fn (incl. a UTF-16 length helper)
- [ ] 2.2 Per-provider limit constants + chunk loop in create/edit delivery execution for slack, telegram, discord, teams, gchat
- [ ] 2.3 WhatsApp migrated to the shared helper; local `splitMessage` deleted
- [ ] 2.4 Slack `formatOutbound` in `extensions/bridges/slack/format.go` (mrkdwn conversion, code protection)
- [ ] 2.5 Telegram `formatOutbound` in `extensions/bridges/telegram/format.go` incl. plain-text fallback rule
- [ ] 2.6 Wire formatters into create/edit delivery execution + progress rendering paths
- [ ] 2.7 Shared conversion corpus (bold/italic/strike/links/code/fences/tables/edge escapes) applied to both Slack and Telegram suites — zero skipped cases
- [ ] 2.8 Provider READMEs note chunking behavior and dialect conversion

## Implementation Details
Reference `_techspec.md` §3.3 (ChunkMessage + dialect formatters), §4 (delete target: WhatsApp `splitMessage`). Chunking and formatting stay provider-side — no daemon↔adapter contract change. Hermes escaping bugs are the known risk (analysis 07 G12) — port tested helper semantics and use Hermes regression suites as oracles. No Slack Block Kit in v1 (text mrkdwn only).

### Relevant Files
- `internal/bridgesdk/chunk.go` — new ChunkMessage + UTF-16 length fn
- `extensions/bridges/{slack,telegram,discord,teams,gchat,whatsapp}/provider.go` — chunk loop in delivery execution
- `extensions/bridges/slack/format.go` — new Slack mrkdwn formatter
- `extensions/bridges/telegram/format.go` — new Telegram MarkdownV2 formatter

### Dependent Files
- `extensions/bridges/*/provider_test.go` — chunking + format corpus cases
- `extensions/bridges/*/provider_delivery_test.go` — over-limit + dialect integration cases
- `extensions/bridges/*/README.md` — chunking + dialect behavior notes

### Competitor References
- `.resources/hermes/gateway/platforms/base.py:5494` — fence-preserving truncation to port
- `.resources/hermes/plugins/platforms/telegram/adapter.py:433,3503,3820-3870,4061` — UTF-16 limits + edit overflow split
- `.resources/hermes/plugins/platforms/slack/adapter.py:1899-1904` — mrkdwn converter
- `.resources/hermes/plugins/platforms/telegram/adapter.py:300-354` — MarkdownV2 escape/fallback
- `.resources/hermes/tests/gateway/test_telegram_format.py:61-119` — regression oracles

## Deliverables
- `bridgesdk.ChunkMessage` + UTF-16 helper, used by all 6 chat providers
- Dialect-correct outbound text on Slack and Telegram across all delivery paths (text, chunks, progress)
- Delete target removed: WhatsApp `splitMessage`
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests
Cases assigned from `_tests.md` (old task_01 + task_07 ownership). Read each case's full definition there before writing tests.

### Unit — Chunking (`internal/bridgesdk/chunk_test.go`)
- [ ] Should split text over the limit on whitespace/newline boundaries with every chunk ≤ limit measured by the injected `lenFn` (D3)
- [ ] Should preserve a fenced code block spanning a boundary: chunk 1 closes the fence, chunk 2 reopens with the same language tag
- [ ] Should measure by UTF-16 code units when given `UTF16Len`: an astral-plane emoji string near the Telegram 4096 cap splits by UTF-16 units, not bytes or runes
- [ ] Should return a single chunk with no `(N/M)` suffix for text under the limit

### Unit — Chunk loop (per provider, `extensions/bridges/<provider>/provider_test.go`, table-driven over the 6 chat providers)
- [ ] Should split a delivery exceeding the provider cap (Slack 40000, Telegram 4096 UTF-16, Discord 2000, Teams 28000, GChat per API, WhatsApp 4096) into N sequential platform sends in order with `(N/M)` markers; the ack carries the last message id
- [ ] Should edit the last chunk and post continuations when an edit overflows the accumulated text (Hermes telegram edit-overflow semantics)

### Unit — Markdown dialects (shared corpus applied to both suites — zero skipped cases)
- [ ] Should convert `**bold** [doc](https://x.y) & <tag>` → `*bold* <https://x.y|doc> &amp; &lt;tag&gt;` for Slack mrkdwn, protecting code spans and fences from conversion (D4)
- [ ] Should escape all Telegram MarkdownV2 specials (incl. `.` and `!`) outside code and fall back to plain text (no `parse_mode`) when escaping would corrupt content — never a Telegram 400
- [ ] Should apply the formatter on EVERY outbound path: text deliveries, chunk continuations, and progress lines — one corpus, three call sites

### Integration (`make test-integration`)
- [ ] Should deliver an over-limit `final` through the fake transport as N sends with zero `permanent` classification errors (suite: `provider_delivery_test.go` contract cases)
- [ ] Should render a delivery with markdown + code fence as dialect-correct bodies on the fake API for Slack AND Telegram, including chunked continuations

Test coverage target: >=80% for touched packages. All tests must pass under the repo gates (`-race` for Go). E2E: Not applicable — provider_delivery contract harness is the highest-fidelity lane for these invariants.

## Success Criteria
- Every assigned test case implemented and passing
- A 6000-char reply delivered to a Discord fake produces 3 ordered sends instead of one API rejection
- `grep -rn "splitMessage" extensions/bridges/whatsapp/` returns nothing
- The shared dialect corpus runs against both providers with zero skipped cases
