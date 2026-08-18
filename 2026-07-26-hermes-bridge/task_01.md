---
status: pending
title: "Progress core: event family, labels/redaction, and ProgressAccumulator"
type: backend
complexity: critical
---

# Task 1: Progress core: event family, labels/redaction, and ProgressAccumulator

## Overview
Ships the progress delivery contract (ToolProgress + projection), friendly tool-label registry with universal redaction, and bridgesdk ProgressAccumulator — the producer side every provider consumes. Bridges today drop ACP tool events at two independent filters; this slice closes that gap end-to-end on the daemon/SDK side so providers can render live tool progress without inventing their own projection or accumulator.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
- MUST add `DeliveryEventTypeProgress` plus a typed `ToolProgress` payload (tool_call_id, tool_id, phase started|completed|failed, label, preview, emoji, duration_ms, monotonic index) per TechSpec §3.1.
- MUST replace the two event-narrowing filters with ONE shared projection helper — delete the double-narrowing, do not patch it (zero-legacy, SD-002).
- MUST gate emission per instance via a typed `delivery_defaults.progress` block (`tool_progress: off|new|all|verbose`, `grouping: accumulate|separate`, `typing`, `reactions`) with resolver precedence per-instance → platform default → global default.
- MUST default `new`+`accumulate` on Slack/Telegram/Discord and `off` on Teams/WhatsApp/GChat (Open Decision #1 recommendation; confirm at execution if contested).
- MUST render `tool.completed` failures as well as successes (divergence from Hermes F3).
- MUST prove the empty-payload contract first (tool_id + phase + index only), then fill `Label`/`Preview`/`Emoji` from the label registry in this same task — no raw tool argument may enter a channel payload before `internal/redact` lands (peer review r2 N-004; techspec §3.8-6).
- MUST enforce the presentation-only invariant: progress events never enter session transcripts/ACP history — encoded as a type boundary and a contract test.
- MUST coalesce progress events under queue saturation (newest-wins per phase) like deltas.
- MUST co-ship contract codegen (OpenAPI + TS types + E2E mocks/matchers) and update all 8 in-tree providers to at least ignore the new payload cleanly (L-025 hard-cut, no dual paths).
- MUST provide a registry mapping tool ID → `{Verb, Connector, DropsPreview, Emoji}` covering every `agh__*` native tool, with a raw fallback (`⚙️ <tool_id>: "<preview>"`) for unknown tools.
- MUST provide per-tool preview builders: shell-aware terminal summarizer (strip pipe-tails, split compound commands), file tools show basename+range (never content), delegate shows "N tasks: …", search tools show the query.
- MUST run every preview/label through ONE canonical secret redactor before it leaves the daemon: promote `internal/modelcatalog/redact.go`'s package-private `secretPatterns`/`RedactString` into a shared leaf helper (e.g. `internal/redact`) consumed by both modelcatalog and toolmeta — do NOT write a second matcher.
- MUST cover the FULL `internal/CLAUDE.md` Security-Invariant taxonomy: raw claim tokens (`agh_claim_*`), MCP auth tokens, OAuth authorization codes, PKCE verifiers, secret-binding refs and `*_secret`-suffixed values, plus provider token shapes (`sk-…`, `xoxb-…`/`xapp-…`, `gh[pousr]_…`, `Bearer …`, and the generic key/value secret forms already in `secretPatterns`). Progress previews are channel messages — the invariant applies verbatim.
- MUST let tool descriptors declare optional `friendly_verb`/`preview` metadata so extension/MCP tools get first-class labels (fixes Hermes F8).
- MUST cap preview length via the resolved display setting (verbose = uncapped).
- MUST encapsulate accumulator state (current bubble id, lines, repeat counter, last-edit timestamp, can-edit flag) in an explicit struct with methods — no shared mutable closures.
- MUST throttle edits (default 1.5s, configurable) and coalesce lines queued inside the window.
- MUST collapse consecutive identical lines to `<line> (×N)`.
- MUST close the current bubble when content lands (reset marker) so the next tool starts a fresh bubble below the answer.
- MUST roll to a continuation bubble before exceeding `maxMessageLen - 64`; only the newest continuation stays mutable.
- MUST support `accumulate` (edit one bubble) and `separate` (one message per line) groupings.
- MUST expose typing-start/stop and reaction (👀/✅/❌) helper hooks driven by progress phases, as no-ops for providers that lack the platform affordance.
- MUST be testable with an injected clock — no wall-clock sleeps in tests.
</requirements>

## Subtasks
- [ ] 1.1 `ToolProgress` type + `DeliveryEventTypeProgress` + projection-event field (`internal/bridges/progress.go` + `delivery_types.go`), with validation
- [ ] 1.2 Single shared projection helper handling text + progress; both old filters deleted; `BridgeDeliveryNotifier` forwards tool events; empty-payload contract proven (tool_id/phase/index only)
- [ ] 1.3 Typed `delivery_defaults.progress` block: schema, validation (`internal/bridges/resource.go`), resolver with per-platform defaults, CLI create/update flags, HTTP contract
- [ ] 1.4 Presentation-only type boundary + transcript-purity contract test
- [ ] 1.5 Label registry + preview builders in `internal/toolmeta` (dedicated package per Open Decision #2; keep files under the 500-line cap)
- [ ] 1.6 Promote the canonical redactor into `internal/redact`; modelcatalog and toolmeta both consume it; update `mage Boundaries`
- [ ] 1.7 Fill `Label`/`Preview`/`Emoji` in the broker projection from toolmeta with redaction applied before payload fill; optional descriptor `friendly_verb`/`preview` metadata surfaced
- [ ] 1.8 `ProgressAccumulator` in `internal/bridgesdk/progress.go` with grouping modes, overflow roll, dedup counter, reset marker
- [ ] 1.9 Typing + reaction hook surface with no-op defaults; godoc on the SDK surface
- [ ] 1.10 Codegen co-ship (OpenAPI, TS types, E2E mock/matcher updates); all 8 providers compile and ignore-or-render the new payload; docs: `packages/site/content/runtime/core/bridges/progress.mdx`

## Implementation Details
Reference `_techspec.md` §3.1 (progress contract), §3.2 (labels/redaction), §3.3 (ProgressAccumulator), §3.7 (`delivery_defaults.progress` block), §3.8 (invariants 1–3, 6–7), §4 (delete targets), §7 (impact audit). Delete targets: the `default:` narrowing in `delivery_broker.go:913-971` and the replay filter switch in `host_api_bridges.go:787-791`, both replaced by the shared helper; package-private redactor copy in `internal/modelcatalog/redact.go` after promotion. Contract change → activate `eng-contract-codegen-coship`. Progress emission runs on the broker's detached pipeline (SD-010). Labels land in the same task after the empty-payload contract is proven — no cross-task staging of Label/Preview fill.

### Relevant Files
- `internal/bridges/progress.go` — new ToolProgress types + projection helper
- `internal/bridges/delivery_types.go` — DeliveryEventTypeProgress + Progress field
- `internal/bridges/delivery_broker.go` — shared projection; delete double-narrowing
- `internal/bridges/resource.go` — typed `delivery_defaults.progress` validation + defaults
- `internal/extension/host_api_bridges.go` — seed/replay path through shared helper
- `internal/extension/bridge_delivery_notifier.go` — forward tool events
- `internal/toolmeta/` — label registry + preview builders (new)
- `internal/redact/` — promoted canonical secret redactor (new)
- `internal/bridgesdk/progress.go` — ProgressAccumulator + typing/reaction hooks
- `internal/cli/bridge.go` — create/update progress flags
- `internal/api/core/bridges.go` — HTTP contract for progress config

### Dependent Files
- `extensions/bridges/*/provider.go` — must handle/ignore `Progress` payloads (L-025)
- `web/src/generated/` + site `api-reference/bridges.mdx` — regenerate via codegen
- `packages/site/content/runtime/core/bridges/progress.mdx` — new progress docs
- `internal/bridges/delivery_projection_test.go` — label-filled payload assertions
- `internal/tools/` — descriptor metadata plumbing for friendly_verb/preview

### Competitor References
- `.resources/hermes/gateway/run.py:16674-17272` — progress pipeline + accumulator drain loop
- `.resources/hermes/gateway/display_config.py:21-160` — tiered defaults + resolver
- `.resources/hermes/agent/display.py:412-626` — builders/labels to port
- `.resources/hermes/gateway/stream_events.py:14-20` — presentation-only contract

## Deliverables
- `progress` delivery family live end-to-end daemon-side (broker → transport), gated per instance
- `internal/toolmeta` + `internal/redact` feeding labeled, redacted payloads
- Unit-testable `ProgressAccumulator` + typing/reaction hooks in bridgesdk
- Delete targets removed: both event-narrowing filters; private modelcatalog redactor copy
- Codegen artifacts regenerated; `progress.mdx` published
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests
Cases assigned from `_tests.md` (old task_02 + task_03 + task_04 ownership). Read each case's full definition there before writing tests.

### Unit — Progress contract & projection (`internal/bridges/delivery_projection_test.go` + `delivery_broker_test.go`)
- [ ] Should project an ACP `tool_call` into exactly one `progress` DeliveryEvent with correct phase and monotonic per-turn index under mode `all`, and emit nothing under mode `off` (D1)
- [ ] Should emit `tool_id` + phase + index with EMPTY `Label`/`Preview`/`Emoji` until the label registry in this task fills them — no raw tool argument ever enters the payload before redaction lands (inv6, peer review r2 N-004)
- [ ] Should dedup consecutive identical tool ids under mode `new` (one progress event, Hermes mode taxonomy)
- [ ] Should map `tool_result` with `is_error=true` to phase `failed` — completion AND failure lines render (divergence from Hermes F3)
- [ ] Should coalesce only same-`(tool_call_id, phase)` items newest-wins under queue saturation, never merge events for DISTINCT tool ids, never drop terminal events, and count forced drops of intermediate lines in `dropped_by_reason` (inv3)
- [ ] Should resolve `delivery_defaults.progress` precedence instance override → platform default → global default, with shipped defaults `new`+`accumulate` on Slack/Telegram/Discord and `off` on Teams/WhatsApp/GChat (Open Decision #1)
- [ ] Should reject invalid `delivery_defaults.progress` values (`tool_progress`, `grouping`, `typing`, `reactions`) at resource validation — typed block, not free-form JSON (P-9)
- [ ] Should keep progress events out of session transcripts and ACP history — type boundary enforced by the transcript-purity contract test (inv1)
- [ ] Should enqueue progress events on the same ordered route queue as text events and deliver exactly one terminal event per delivery (inv2)
- [ ] Should project the seed/replay path and the live notifier path through the ONE shared helper with identical output — both retired narrowing filters are gone, not patched (D1 delete target; suite: `bridge_delivery_notifier_test.go`)
- [ ] Should yield the SAME `transcript.MarshalAgentEvent` fingerprint for a `tool_call` projected via live and via seed/replay, delivered exactly once — a late-registering delivery never double-posts progress history (peer review N-006)

### Unit — Labels, previews & redaction (`internal/toolmeta`, `internal/redact`, `delivery_projection_test.go`)
- [ ] Should map every registered `agh__*` tool to its curated `{Verb, Connector, Emoji}` table-driven, and render unknown tools as the raw fallback `⚙️ <tool_id>: "<preview>"`
- [ ] Should summarize terminal commands shell-aware: strip pipe-tails, split compound commands (`ls -la | head -5 && git status` fixture)
- [ ] Should render file-tool previews as basename+range only — `write_file` args containing `content` never appear in a preview
- [ ] Should redact provider token shapes in previews — `Bearer …`, `sk-…`, `xoxb-…`/`xapp-…`, `gh[pousr]_…` → `[REDACTED]` (inv6)
- [ ] Should redact the full Security-Invariant taxonomy embedded in a `terminal.command` preview: `agh_claim_*` tokens, MCP auth tokens, OAuth authorization codes, PKCE verifiers, secret-binding refs and `*_secret`-suffixed values (inv6, peer review B-001)
- [ ] Should prove modelcatalog and toolmeta consume the SAME canonical pattern set from `internal/redact` — a drift test fails if either forks the list (one source of truth)
- [ ] Should cap preview length per the resolved display setting and leave `verbose` uncapped
- [ ] Should surface descriptor-declared `friendly_verb`/`preview` metadata so extension/MCP tools get first-class labels (fixes Hermes F8; descriptor schema/digest surfaces re-checked)
- [ ] Should fill `Label`/`Preview`/`Emoji` in the broker projection from toolmeta with redaction applied before payload fill

### Unit — ProgressAccumulator (`internal/bridgesdk/progress_test.go`; injected clock — zero `time.Sleep`)
- [ ] Should batch three progress lines arriving inside one throttle window into exactly one edit carrying all three
- [ ] Should collapse identical consecutive lines to `<line> (×N)`
- [ ] Should open a fresh bubble (new post, not edit) for the next line after content lands (reset marker) so progress never edits above the answer
- [ ] Should roll to a continuation bubble before exceeding `maxMessageLen - 64` and freeze the earlier bubble (only the newest continuation stays mutable)
- [ ] Should post one message per line with zero edits under `separate` grouping
- [ ] Should fire typing-start on phase `started` and the ❌ reaction hook on phase `failed`; providers with no-op hooks proceed without error
- [ ] Should defer an edit arriving at t=0.5s to t=1.5s under the injected-clock throttle (default 1.5s, configurable)

### Integration (`make test-integration`)
- [ ] Should deliver an ordered `start → progress×N → final` turn to the fake adapter end-to-end while the transcript store contains ZERO progress entries (inv1, inv2; suite: `bridge_delivery_integration_test.go`)
- [ ] Should deliver a progress event whose label matches the toolmeta registry and whose preview is redacted — proving the daemon-renders/adapter-delivers split and that no raw secret crosses the daemon→adapter transport (inv6)

### E2E — Runtime (`make test-e2e-runtime`, bridge contract lane)
- [ ] Should pass the contract lane with mocks/matchers updated for the `progress` event family — contract co-ship green (L-007)

Test coverage target: >=80% for touched packages. All tests must pass under the repo gates (`-race` for Go).

## Success Criteria
- Every assigned test case implemented and passing
- Coverage >=80% on touched packages
- A tool-heavy fake turn delivers progress events to the fake adapter; a fake `agh__terminal` call with an embedded token renders as a redacted labeled preview in the delivered payload
- Accumulator suite runs with zero `time.Sleep` (injected clock proven)
- `codegen-check` passes; both event-narrowing filters deleted
