# Bridge Usability Parity (hermes-bridge) — Test Plan

> **What this is.** The complete, numbered test-case inventory for the hermes-bridge program,
> the companion to `_techspec.md`. The TechSpec's §9 "Verification Strategy" summarizes the
> lanes and points here; this file is the authoritative case list the executors of tasks 01–08
> work through, and the E2E backbone the QA pair (tasks 09–10, seeded by `_qa.md`) maps
> journeys onto. Every case names its owning suite, its root-cause/invariant reference, and the
> task that lands it.

## Conventions (house style — `eng-test-conventions` + `eng-consolidate-test-suites`)

- Cases are written `Should …` to match `t.Run("Should …")`; table-driven where a fixture set
  applies (dialect corpus, provider matrix, redaction taxonomy).
- **Owning layer is named per case.** Build into the canonical suite for that layer; do not
  create duplicate/standalone regression files. Layers:
  - `internal/bridges` — `delivery_broker_test.go`, `delivery_projection_test.go`,
    `types_test.go` (contract, projection, coalescing, ledger checkpoints).
  - `internal/toolmeta` + `internal/redact` — new package suites (labels, previews, redaction).
  - `internal/bridgesdk` — `chunk_test.go`, `progress_test.go`, provider-runtime suites.
  - `internal/extension` — `bridge_delivery_notifier_test.go`,
    `bridge_delivery_integration_test.go`, the `host_api_bridges` ingest suite,
    `provider_conformance_matrix_integration_test.go`, `bridge_docs_conformance_test.go`.
  - `internal/cli` — `bridge_test.go` (+ `cli_integration_test.go`); `internal/doctor` —
    `doctor_test.go`.
  - `internal/store/globaldb` — canonical migration suite; `internal/daemon` — boot tests.
  - Providers — `extensions/bridges/<provider>/provider_test.go` +
    `provider_delivery_test.go` (contract cases over the fake transport / fake platform API).
  - E2E runtime — `make test-e2e-runtime` bridge contract lane (mock + matchers co-ship, L-007).
  - Web — vitest via `bunx turbo run test --filter=./web` (logic only, no snapshot filler);
    Playwright via `make test-e2e-web`.
- Build tags: `//go:build integration` for integration; E2E lanes per their make targets.
  `-race` / `CGO_ENABLED=1` on broker/accumulator/ledger concurrency cases.
- **Hermetic tests only** — no network. Platform APIs are faked (recorded-request fakes);
  vendor schemas are committed fixtures with source+version noted in the header.
- **HTTP cases assert status code AND body**, never status-only. New routes co-ship
  OpenAPI + TS types + E2E mocks (`eng-contract-codegen-coship`).
- 80% coverage floor per touched package. `mage Boundaries` green whenever a package is added.
- References: `D1..D9` = `_techspec.md §2` root causes; `inv1..inv9` = `_techspec.md §3.8`
  safety invariants; `task_NN` = owning task; Hermes `F*`/`G*`/`C*` = analysis findings.

---

## 1. Unit — Daemon core (`internal/bridges`, `internal/extension`, `internal/toolmeta`, `internal/redact`)

Progress contract & projection (task_01; suites: `delivery_projection_test.go` +
`delivery_broker_test.go`):
1. Should project an ACP `tool_call` into exactly one `progress` DeliveryEvent with correct
   phase and monotonic per-turn index under mode `all`, and emit nothing under mode `off` (D1).
2. Should emit `tool_id` + phase + index with EMPTY `Label`/`Preview`/`Emoji` until the
   toolmeta fill lands within task_01 — no raw tool argument ever enters the payload before
   redaction (inv6, peer review r2 N-004).
3. Should dedup consecutive identical tool ids under mode `new` (one progress event, Hermes
   mode taxonomy).
4. Should map `tool_result` with `is_error=true` to phase `failed` — completion AND failure
   lines render (divergence from Hermes F3).
5. Should coalesce only same-`(tool_call_id, phase)` items newest-wins under queue saturation,
   never merge events for DISTINCT tool ids, never drop terminal events, and count forced
   drops of intermediate lines in `dropped_by_reason` (inv3).
6. Should resolve `delivery_defaults.progress` precedence instance override → platform default
   → global default, with shipped defaults `new`+`accumulate` on Slack/Telegram/Discord and
   `off` on Teams/WhatsApp/GChat (Open Decision #1).
7. Should reject invalid `delivery_defaults.progress` values (`tool_progress`, `grouping`,
   `typing`, `reactions`) at resource validation — typed block, not free-form JSON (P-9).
8. Should keep progress events out of session transcripts and ACP history — type boundary
   enforced by the transcript-purity contract test (inv1, task_01.4).
9. Should enqueue progress events on the same ordered route queue as text events and deliver
   exactly one terminal event per delivery (inv2).
10. Should project the seed/replay path and the live notifier path through the ONE shared
    helper with identical output — both retired narrowing filters are gone, not patched
    (D1 delete target; suite: `bridge_delivery_notifier_test.go`).
11. Should yield the SAME `transcript.MarshalAgentEvent` fingerprint for a `tool_call`
    projected via live and via seed/replay, delivered exactly once — a late-registering
    delivery never double-posts progress history (peer review N-006).

Labels, previews & redaction (task_01; suites: `internal/toolmeta`, `internal/redact`,
`delivery_projection_test.go`):
12. Should map every registered `agh__*` tool to its curated `{Verb, Connector, Emoji}`
    table-driven, and render unknown tools as the raw fallback `⚙️ <tool_id>: "<preview>"`.
13. Should summarize terminal commands shell-aware: strip pipe-tails, split compound commands
    (`ls -la | head -5 && git status` fixture).
14. Should render file-tool previews as basename+range only — `write_file` args containing
    `content` never appear in a preview.
15. Should redact provider token shapes in previews — `Bearer …`, `sk-…`, `xoxb-…`/`xapp-…`,
    `gh[pousr]_…` → `[REDACTED]` (inv6).
16. Should redact the full Security-Invariant taxonomy embedded in a `terminal.command`
    preview: `agh_claim_*` tokens, MCP auth tokens, OAuth authorization codes, PKCE verifiers,
    secret-binding refs and `*_secret`-suffixed values (inv6, peer review B-001).
17. Should prove modelcatalog and toolmeta consume the SAME canonical pattern set from
    `internal/redact` — a drift test fails if either forks the list (one source of truth).
18. Should cap preview length per the resolved display setting and leave `verbose` uncapped.
19. Should surface descriptor-declared `friendly_verb`/`preview` metadata so extension/MCP
    tools get first-class labels (fixes Hermes F8; descriptor schema/digest surfaces
    re-checked).
20. Should fill `Label`/`Preview`/`Emoji` in the broker projection from toolmeta with
    redaction applied before payload fill (task_01.4).

Inbound edit family (task_06; suite: `internal/bridges/types_test.go`):
21. Should validate the `edit` family payload (edited message id, new text, original
    timestamp), enforce mutual exclusion with message/command/action/reaction families, and
    round-trip envelope `ReplyToText`/`ReplyToAuthorID`/`ReplyToAuthorName` through JSON (D9).

Durable ledger checkpoints (task_06; suite: `delivery_broker_test.go`):
22. Should checkpoint on register (row `active`), advance `last_sent_seq`/`last_acked_seq` +
    `remote_message_id` on ack, and mark terminal transitions — while a delta storm produces
    ZERO per-delta writes (write-amplification guard asserted via store call counts; inv4).
23. Should keep `DeliveryMetrics` counters durable — persisted alongside or exactly derivable
    from the ledger rows (D8).

---

## 2. Unit — bridgesdk (`internal/bridgesdk`)

Chunking (task_02; suite: `chunk_test.go`):
1. Should split text over the limit on whitespace/newline boundaries with every chunk ≤ limit
   measured by the injected `lenFn` (D3).
2. Should preserve a fenced code block spanning a boundary: chunk 1 closes the fence, chunk 2
   reopens with the same language tag.
3. Should measure by UTF-16 code units when given `UTF16Len`: an astral-plane emoji string
   near the Telegram 4096 cap splits by UTF-16 units, not bytes or runes.
4. Should return a single chunk with no `(N/M)` suffix for text under the limit.

ProgressAccumulator (task_01; suite: `progress_test.go`; injected clock — zero `time.Sleep`):
5. Should batch three progress lines arriving inside one throttle window into exactly one edit
   carrying all three.
6. Should collapse identical consecutive lines to `<line> (×N)`.
7. Should open a fresh bubble (new post, not edit) for the next line after content lands
   (reset marker) so progress never edits above the answer.
8. Should roll to a continuation bubble before exceeding `maxMessageLen - 64` and freeze the
   earlier bubble (only the newest continuation stays mutable).
9. Should post one message per line with zero edits under `separate` grouping.
10. Should fire typing-start on phase `started` and the ❌ reaction hook on phase `failed`;
    providers with no-op hooks proceed without error.
11. Should defer an edit arriving at t=0.5s to t=1.5s under the injected-clock throttle
    (default 1.5s, configurable).

Provider runtime (task_07; suites: new runtime files' tests):
12. Should drive the hoisted lifecycle initialize→ready reporting, ownership-sync retry, and a
    clean shutdown drain (D7).
13. Should produce the SAME reconcile state transitions (instance-config add/remove/
    path-conflict, route-map swap) the retired per-provider copies produced — the strongest
    existing per-provider cases ported here as the canonical home.

---

## 3. Unit — CLI & doctor (`internal/cli`, `internal/doctor`)

Slack manifest (task_04; suite: `bridge_test.go`):
1. Should emit a manifest that validates against the VENDORED Slack app-manifest JSON schema
   fixture (no network; fixture header notes source + version).
2. Should hold the scope↔event RELATIONS: every subscribed `message.*`/`app_mention`/
   `reaction_added` event has its required read scope; `chat:write` always present; reaction
   events ↔ `reactions:write` (invariant, not snapshot — analysis 04 §3.5).
3. Should template request URLs from the instance webhook path and set
   `socket_mode_enabled: false` (AGH is webhook-based).
4. Should write the file under `--write` (with `--out PATH` override) and print the exact
   "paste into api.slack.com → From an app manifest" epilogue; a bare invocation emits valid
   JSON on stdout only (agent-manageable, SD-011).

Setup wizards (task_04; suite: `bridge_test.go`):
5. Should reject a phone number pasted into `phone_number_id` with the "look below the From
   dropdown" remediation, name the product on wrong-product pastes (`sk-…` → OpenAI key,
   `xoxb-…`, `ghp_…`), and pass valid WhatsApp field shapes (validators ported from
   `setup_whatsapp_cloud.py:52-157`).
6. Should accept a valid BotFather token against `^\d+:[A-Za-z0-9_-]{30,}$` and reject
   truncated ones.
7. Should auto-generate `verify_token`/`webhook_secret` when absent and keep existing values
   as masked defaults on re-run (inv9).
8. Should complete `--json` mode with zero prompts: full config in → instance created +
   secrets bound + structured result out (SD-011).
9. Should build the Discord invite URL with the required scopes and permission bits from the
   instance config.

Verify / send-test / doctor (task_04; suites: `bridge_test.go`, `doctor_test.go`):
10. Should run bridge probes only under `agh doctor --only bridge`; a failing identity probe
    yields status `fail` with a non-empty remediation naming the secret slot.
11. Should render `verify` as structured JSON with per-check `{check, status, remediation}`
    records and exit non-zero on any fail (records, not log lines — analysis 02 §5).
12. Should route `send-test` through the REAL delivery path (fake transport records a platform
    send) while `test-delivery` still performs zero platform calls (D2).

---

## 4. Unit — Providers (`extensions/bridges/<provider>/provider_test.go`)

Chunk loop (task_02; per provider, table-driven over the 6 chat providers):
1. Should split a delivery exceeding the provider cap (Slack 40000, Telegram 4096 UTF-16,
   Discord 2000, Teams 28000, GChat per API, WhatsApp 4096) into N sequential platform sends
   in order with `(N/M)` markers; the ack carries the last message id.
2. Should edit the last chunk and post continuations when an edit overflows the accumulated
   text (Hermes telegram edit-overflow semantics).

Progress rendering — Slack/Telegram/Discord (task_03; per provider):
3. Should render `ToolProgress{phase: started}` as a platform post whose body carries the
   labeled line and the inbound `thread_ts` (Slack `chat.postMessage`; Telegram
   `sendMessage`; Discord create) — thread routing keeps channels readable.
4. Should render a second progress event inside the window as an edit with both lines against
   the SAME message id (`chat.update` / `editMessageText` / Discord edit).
5. Should render phase `failed` as a ❌ line with a short error and a ≥30s `completed` as
   ✅ + duration (completion lines — Hermes F3 divergence).
6. Should make zero platform API calls for progress events on a mode-`off` instance.
7. Should start typing on the first progress event (Slack `assistant.threads.setStatus`,
   Telegram `sendChatAction`, Discord typing) and clear it when the final answer posts.
8. Should honor a 429 `Retry-After` on edits with backoff and no dropped event (rate-limit
   discipline, G10/G11).
9. Should execute Slack `chat.delete` against the fake API (capability for a future
   `cleanup_progress`; default-off — Hermes F5).

Progress rendering — Teams/GChat/WhatsApp + issue bridges (task_03; per provider):
10. Should make zero progress platform calls on the default (off-tier) instance config.
11. Should render progress on an opted-in Teams instance as activity create then update with
    `TextFormat: markdown`.
12. Should render progress on an opted-in GChat instance as message create then patch.
13. Should render two DISTINCT tool ids on an opted-in WhatsApp instance as two separate
    status posts and collapse identical consecutive ids into one (`separate` grouping).
14. Should ack a delivery request carrying a `Progress` payload on GitHub/Linear without any
    platform side effect (issue bridges stay progress-free but contract-clean; inv7).

Markdown dialects (task_02; shared corpus applied to both suites — zero skipped cases):
15. Should convert `**bold** [doc](https://x.y) & <tag>` →
    `*bold* <https://x.y|doc> &amp; &lt;tag&gt;` for Slack mrkdwn, protecting code spans and
    fences from conversion (D4).
16. Should escape all Telegram MarkdownV2 specials (incl. `.` and `!`) outside code and fall
    back to plain text (no `parse_mode`) when escaping would corrupt content — never a
    Telegram 400.
17. Should apply the formatter on EVERY outbound path: text deliveries, chunk continuations
    (task_02), and progress lines (task_03) — one corpus, three call sites.

Inbound edits & reply context (task_06; per provider):
18. Should map a Slack `message_changed` webhook to an `edit`-family envelope with the new
    text and original message id (D9).
19. Should fill reply-to context on a Slack threaded reply from the bounded parent-text cache;
    a cache miss degrades to empty fields with NO fetch storm.
20. Should map a Telegram `edited_message` update to an `edit`-family envelope.

Verify probes (task_04; suite: `slack/provider_test.go`):
21. Should report a fake `auth.test` missing `mpim:history` as a check result naming the
    missing scope and the reinstall requirement, and a user token (no `bot_id`) as
    "user token, expected bot token" (scope diff vs the task_04 manifest set).

---

## 5. Integration (`make test-integration`; fake transport / real store / real daemon wiring)

1. Should apply the `bridge_deliveries` migration at the registry tail with the checksum chain
   intact — append-only proof (task_06; suite: `internal/store/globaldb` canonical migration
   suite; `eng-schema-migration`, L-021).
2. Should deliver an ordered `start → progress×N → final` turn to the fake adapter end-to-end
   while the transcript store contains ZERO progress entries (inv1, inv2; task_01; suite:
   `bridge_delivery_integration_test.go`).
3. Should deliver a progress event whose label matches the toolmeta registry and whose preview
   is redacted — proving the daemon-renders/adapter-delivers split and that no raw secret
   crosses the daemon→adapter transport (inv6; task_01).
4. Should render an opted-in low-tier (Teams/GChat/WhatsApp) turn as ordered status messages
   over the fake transport and still deliver the final answer (task_03).
5. Should render a delivery with markdown + code fence as dialect-correct bodies on the fake
   API for Slack AND Telegram, including chunked continuations (tasks 01+07 interplay).
6. Should deliver an over-limit `final` through the fake transport as N sends with zero
   `permanent` classification errors (task_02; suite: `provider_delivery_test.go` contract
   cases).
7. Should round-trip `agh bridge manifest slack` against a daemon with the Slack extension
   installed and return the manifest for a real instance id (task_04; suite:
   `cli_integration_test.go`).
8. Should complete the wizard `--json` round-trip against a daemon: instance exists, bindings
   listed, Telegram `setWebhook` hit the fake platform endpoint — or was printed under
   `--print-only` (task_04).
9. Should register the Telegram webhook via `POST /api/bridges/:id/webhook/register` with
   ZERO CLI involvement — HTTP-only agent parity proof, status + body asserted (SD-011;
   task_04; peer review r2 N-003).
10. Should aggregate verify checks through the daemon→provider→daemon round-trip WITHOUT
    changing instance lifecycle state (inv8; task_04).
11. Should get an answer to the check request from ALL 8 in-tree providers: identity-capable
    ones return real probe results, the rest return explicit `skipped` — none silently absent
    (inv7, L-025 hard-cut; task_04).
12. Should render an ingested `edit` envelope as a distinct "edited" prompt block and surface
    reply-to context in the rendered prompt (task_06; suite: `host_api_bridges` ingest suite).
13. Should recover a mid-stream delivery after a simulated restart (fresh broker over the same
    store): resume reconciles the fake adapter to the same `remote_message_id`, OR fail-open
    posts the standard "session stopped" terminal error — the channel is never left
    half-streamed silently (inv4; task_06).
14. Should keep delivery metrics across the restart (task_06).
15. Should complete boot reconciliation BEFORE the broker accepts new delivery registrations
    (inv5, SD-008 composition-root ordering; suite: `internal/daemon` boot tests).
16. Should auto-discover every `extensions/bridges/*` provider in the conformance matrix: a
    synthetic provider directory with a valid manifest is discovered and passes; removing a
    required secret slot fails the matching assertion; all 8 real providers pass
    initialize/health round-trips on the fake transport (task_07; suite:
    `provider_conformance_matrix_integration_test.go`).
17. Should enforce docs↔code drift as INVARIANTS: the `index.mdx` slots table covers every
    discovered provider, the docs Slack scope list matches the task_04 scope constants, and
    every provider has a setup section — adding provider+docs together stays green, adding one
    side fails (task_07 drift suite; task_08 lands the docs that make it green; suite:
    `bridge_docs_conformance_test.go`).
18. Should co-ship OpenAPI: every new bridge route (`GET …/providers/slack/manifest`,
    `POST …/:id/verify`, `POST …/:id/send-test`, `POST …/:id/webhook/register`) is present in
    the generated contract + generated TS types, and each endpoint asserts status AND body
    (`eng-contract-codegen-coship`).
19. Should isolate `bridge_deliveries` by `scope`/`workspace_id` (indexed, no instance join):
    workspace A never lists/queries workspace B's rows through metrics or reconcile paths
    (techspec §3.7/§7 workspace audit; task_06; peer review N-003).
20. Should keep all 8 provider suites green with behavior assertions UNTOUCHED after the
    scaffolding hoist (wiring-only diffs — the suites are the regression oracle), conformance
    matrix green, `mage Boundaries` green (task_07).

---

## 6. E2E — Runtime (`make test-e2e-runtime`, bridge contract lane)

1. Should pass the contract lane with mocks/matchers updated for the `progress` event family —
   contract co-ship green (L-007; task_01).
2. Should ack progress events in order through the bridge lane for a tool-heavy turn
   (task_03).
3. Should round-trip an opted-in low-tier provider through the updated contract mocks
   (task_03).
4. Should cover the verify and send-test request/response shapes in the contract mocks
   (task_04).
5. Should cover the inbound `edit` family + reply-to envelope fields in the contract mocks
   (task_06).
6. Should exercise boot reconciliation in the daemon-restart scenario: restart mid-turn, then
   resume-or-fail-open observed at the fake adapter (task_06).
7. Should keep the contract lane green after the bridgesdk provider-runtime migration
   (task_07).

---

## 7. E2E — Web (`make test-e2e-web`, Playwright; mocked daemon contract)

Behavior flows, not pixels — visual parity is `eng-ui-screenshot`'s job (task_05.5).

1. Bridges create flow — Should open the create wizard, select a provider WITH a manifest
   endpoint (Slack), surface the manifest step (copy + deep link), complete creation, and show
   the state-aware setup checklist on the detail panel (task_05).
2. Verify flow — Should run the Verify action from the detail panel and render per-check
   status inline on the matching secret-binding cards, including a failure remediation
   (task_05).
3. Send-test vs dry-run — Should expose "Send test" beside the existing dry-run
   test-delivery with distinguishing labels, and reflect the send-test result (SD-007
   truthful-UI; task_05).

---

## 8. Web Unit / Component (`bunx turbo run test --filter=./web`, vitest)

Logic/behavior only — no snapshot/render-shape filler.

1. Should render the manifest step for a Slack provider selection and copy the fetched JSON;
   the step is ABSENT for providers without a manifest endpoint (task_05).
2. Should render per-check status on the matching secret card on verify-mutation success and
   the remediation text on failure (task_05).
3. Should derive every checklist item from mocked bridge state (unbound secret → unchecked
   with CTA; enabled+healthy → all checked) — items map 1:1 to daemon-observable facts, no
   invented controls (SD-007; task_05).
4. Should serialize the progress fields (mode/grouping/typing/reactions) into
   `delivery_defaults.progress` on both create and edit (task_05).
5. Should produce the correct POST body from the full create-wizard MSW flow with manifest
   step + progress fields (task_05 integration).

---

## Deferred / Out of Scope (no v1 test)

- Outbound media upload (analysis 07 C9/R10) and Slack Block Kit rendering (text mrkdwn only
  in v1 — task_02).
- GChat per-user OAuth (C14) and the GChat CardV2 typing card (needs a card abstraction —
  task_03 divergence note).
- Telegram QR managed-bot onboarding (rejected — third-party broker, C3); Discord voice
  (rejected — C10); startup auto-manifest emission (rejected — C15).
- In-channel pairing-code issuer (deferred — `dm_policy` + `paired_user_ids` cover the ACL
  need, analysis 07 §3.8-5).
- Full-text delivery audit ledger — v1 is checkpoint-only (Open Decision #5); progress events
  are NOT checkpointed (text stream checkpoints only).
- Discord inbound message-edit routing IF the interactions surface lacks the event
  (UNCONFIRMED — task_06.4 verifies during implementation and skips with evidence).
- Real-platform sends in CI — no platform credentials in-tree; the fake-transport contract
  harness is the highest-fidelity automated lane, and REAL sends are exercised manually in the
  QA cycle (tasks 09–10 via `_qa.md`).
