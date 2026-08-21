---
status: completed
title: "P6b — View Programs: view.provider Runtime, TS SDK, React Kit"
type: backend
complexity: critical
---

# Task 8: P6b — View Programs: view.provider Runtime, TS SDK, React Kit

## Overview

Delivers programmable extension views end-to-end: the `view.provider` capability on the provides-RPC channel (`view/open|event|close` + host-API `view/patch` → SSE fan-out), the daemon `ViewService` session runtime (per-client sessions, causal generations, event admission/caps, three-band budgets, circuit-breaking), the subprocess protocol additions shared by all provides (cancel frames, per-request contexts/`AbortSignal`, negotiated view timeout, view rate class, in-flight caps), the TS SDK `view-provider.ts` registrar + session registry, and the new **`@compozy/extension-react`** workspace package at `sdk/react/` (component kit + persistent-mode reconciler + hooks + starvation guard). Ships the TypeScript fixture extension and deletes the dead `execute_hook` daemon-request seam.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting; read every ADR in `adrs/` and any `memory/task_NN.md` present
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
- NO WORKAROUNDS — fix root causes; frozen `_dx.md`/Part II contracts are binding; surface gaps as clarifications, never invent
</critical>

<requirements>
1. MUST add `view.provider` as the 8th provides capability (`internal/extensionprotocol/capabilities.go`: const + `capabilityServiceMethods` entry for `view/open|event|close`) and run `make codegen` so both SDK contract tables, conformance fixtures, and all validation points propagate; the handshake's method-coverage enforcement follows automatically; `DescribePayload` gains the static view-metadata field (daemon lands first — strict describe decoding); initialize response advertises view ids (`watch_source_kinds` pattern).
2. MUST implement the daemon runtime as `ViewService` (single authority, composition-root wired — every open/event/stream/close/effect transition routes through it): `OpenSession` binds the caller's authenticated windowmanager client identity + canonical workspace, mints opaque per-client sessions + `stream_token`; `AdmitEvent` enforces seq admission, 1 in-flight coalescible with cancel-of-superseded, ≤4 independent action events (`view_busy`), and stamps daemon-assigned **causal generations**; frames/patches/effects from canceled or superseded generations are rejected even on the `InReplyTo=0` push path (SI-19); late frames dispose by session-id validity (SI-20); teardown is idempotent, two-way, and fires on client disconnect and palette dismiss (SI-13).
3. MUST enforce session-scoped, token-authorized streams server-side: `POST /views/{id}/open` (401/404/422 per `_dx.md`), `GET /view-sessions/{s}/stream`, `POST /view-sessions/{s}/events` (202/409/410/403), `DELETE /view-sessions/{s}` — ownership of workspace/client/session checked on every call; frames for other sessions never reach a socket; both transports + OpenAPI/TS + parity rows.
4. MUST implement `view/patch` as a host-API method (catalog entry + handler + permission wiring + rate class) fanning out to the session stream; effects are the typed closed union, **at-most-once by effect id** (`ack_effects` fence; resync re-delivers frames WITHOUT acked effects; `pick_files` results correlate by effect id — SI-21).
5. MUST implement the degradation contract host-side (web, on task_06's shell): event counters on controlled values (stale echoes never revert typing — SI-14), handler-id quarantine two frames (SI-15), three-band budget (0 ms host-local · ~150 ms busy-with-previous-rows · 3 s degraded+retry), circuit-break at 3 consecutive misses, crash → unavailable frame; Esc/⌫/navigation and sibling views stay live throughout (SI-16); `complete:true` flips to local filtering.
6. MUST land the subprocess protocol additions for BOTH peers and the Go SDK where shared: cancel frames + per-request `context.Context`/`AbortSignal` in `TransportHandler`, negotiated `default_view_timeout_ms`, view-class rate limiter distinct from the host-API bucket, per-extension in-flight caps — without breaking existing provides (conformance suites extended).
7. MUST build `@compozy/extension-react` at `sdk/react/` as a first-class workspace package: the frozen kit inventory from `_dx.md` § "The React kit at a glance" (components, handler-valued props → stable opaque ids, hooks incl. `useNavigation`/`usePromise`/`useCachedPromise`/`useCachedState`/`showToast`), react-reconciler persistent mode → frames + frame-clock-batched content-hash-deduped patches, kit-level validation (`destructive ⇒ confirmation`), starvation guard warning. Wiring is explicit: root `package.json` workspaces entry, `scripts/gate.sh` `classify()` case for `sdk/react/*`, root vitest `projects` entry, oxlint override opt-in (react-doctor set), tsconfig chain + `postbuild.mjs` + LICENSE mirroring `sdk/typescript`, 500-line cap per file.
8. MUST extend `sdk/typescript`: `view-provider.ts` registrar (inheriting `ExtensionProvideSurfaces` rollback), the view-session registry, `AbortSignal` in `TransportHandler`; scaffold template `view-provider-ts`; typing freshness rides `codegen-check` + Bun typecheck (a Go vocabulary change that breaks the kit fails the gate).
9. MUST ship the TypeScript fixture extension in `internal/extension/testdata/` (program view with search/chips/detail/form/destructive + slow/crash modes) and complete program-specific validation: `program: true` in a Go-toolchain extension fails validate with the exact `_dx.md` message; `program: true` without `view.provider` implemented fails build describe-coverage (UT-167/IT-030).
10. MUST delete the `execute_hook` **daemon-request seam** with precision: only the declared-but-never-called extension→daemon request registration falls (ADR-008 — `view/event` is the interactive path); the live daemon→extension hook dispatch (`manager_bridge_resources.go` service call, `hook-ts` scaffold, testing harness) is NOT this seam — verify call direction before deleting; no dual seams remain.
11. MUST co-ship docs: `packages/site/content/docs/extensions/develop.mdx` gains the View-providers provide-surface section; React-kit authoring docs; glossary `view.provider` provide-surface entry; `skills/compozy/` update.
12. MUST block and surface if the cited artboard is missing at execution time.
</requirements>

## Visual Contract

| ID    | Reference artifact + state | Implementation target + state | Viewport | Fidelity | Authorized differences + authority |
| ----- | -------------------------- | ----------------------------- | -------- | -------- | ---------------------------------- |
| VC-01 | `docs/design/opendesign/command-palette/command-palette-view-bands.html` — busy band (previous rows + busy indicator) | TS fixture in slow mode past the soft budget | 1440×900 | normative | None |
| VC-02 | `command-palette-view-bands.html` — degraded band (last-good + inline retry) | fixture past the hard ack | 1440×900 | normative | None |
| VC-03 | `command-palette-view-bands.html` — circuit-broken state | third consecutive miss | 1440×900 | normative | None |
| VC-04 | `command-palette-view-bands.html` — crashed unavailable frame naming the extension | fixture crash mode | 1440×900 | normative | None |
| VC-05 | `command-palette-view-bands.html` — "view reloaded" note | `extension dev` edit with the view open | 1440×900 | normative | None |

Evidence for each row: `.compozy/tasks/command-palette/evidence/visual/task_08/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 8.1 Capability + protocol: `view.provider` entry, service methods, `make codegen` propagation, DescribePayload view metadata, initialize advertisement
- [x] 8.2 Subprocess protocol additions (cancel frames, per-request ctx/AbortSignal, negotiated timeout, rate class, in-flight caps) across daemon transport + Go SDK + TS SDK, with conformance coverage
- [x] 8.3 `internal/extension/view_source.go` service caller (standard gate) + `ViewService` session runtime in `internal/cmdpalette` (registry, generations, admission, caps, teardown, instance invalidation)
- [x] 8.4 `view/patch` host-API method (catalog + handler + permission + rate) + session-scoped SSE fan-out
- [x] 8.5 View-session routes on both transports (open/stream/events/close per `_dx.md` status codes) + OpenAPI/TS + parity rows
- [x] 8.6 Web host runtime on the task_06 shell: event counters, quarantine, bands, circuit-break, `complete:true` local filter, effect ack loop
- [x] 8.7 `sdk/typescript`: `view-provider.ts` + session registry + `AbortSignal`; scaffold template `view-provider-ts`
- [x] 8.8 `sdk/react` package: kit components/hooks + persistent-mode reconciler + patch batching/dedupe + starvation guard + full workspace/gate/lint wiring
- [x] 8.9 TypeScript fixture extension (browser view + slow/crash modes) + program-specific validation (Go-toolchain rejection, describe-coverage)
- [x] 8.10 Precise `execute_hook` daemon-request seam deletion (direction-verified) — no dual seams
- [x] 8.11 Docs (develop.mdx View providers, React-kit authoring, glossary `view.provider`) + `skills/compozy/`
- [ ] 8.12 Visual Contract evidence bundles VC-01..05 — capture remains in the accepted task_12 QA tail

## Implementation Details

Reference `_spec.md` Part II: Core Interfaces (`ViewService`, `ViewOpenRequest`/`ViewEvent`/`ViewFrame`), Key Decisions (attached clients — token-proven self-originated opens), Assumptions (band budgets, resource caps), Safety Invariants 13–21; `analysis/08` seam tables; `analysis/05`/`06` event-loop references.

### Skills

`golang-master` · `eng-code-guidelines` · `eng-cleanup-failure-paths` · `eng-contract-codegen-coship` · `eng-test-conventions` · `testing-boss` · `vitest` · `eng-ui-screenshot` · `documentation-writer`

### Relevant Files

- `internal/extensionprotocol/capabilities.go` — the closed provides set (7 members; `view.provider` is the 8th) + `capabilityServiceMethods`
- `internal/subprocess/transport.go` + `handshake.go` (method-coverage enforcement at L257) + `process*.go` + `health.go` — the channel gaining cancel/timeout/rate/caps
- `internal/extension/tool_runtime.go:158-226` (`extensionServiceProcessForInstance`) — the exact seam `view_source.go` calls
- `internal/extension/watch_source.go` + `sdk/typescript/src/{connectivity-provider.ts,forge-provider.ts,extension-provide-surface.ts,capabilities.ts,transport.ts,watch-source.ts}` — the registrar/kind/transport patterns the view-provider work extends (anchors in `analysis/08`)
- `internal/codegen/hostapi/catalog.json` + `generate.go` + `internal/extension/host_api_methods.go` + `host_api_dispatch.go` + `host_api_rate_limit.go` — host-API registration for `view/patch`
- `internal/daemon/clarify_bridge.go` + `clarify_bridge_lifecycle.go` — the per-invocation session runtime the view-session registry generalizes (opaque id, owner check, cancel propagation, `OnceFunc` release)
- `internal/extension/manager_bridge_resources.go:77` + `internal/extension/scaffold_templates/hook-ts/` + `sdk/typescript/src/testing/harness.ts:52` — the LIVE `execute_hook` hook-dispatch usages that must survive the seam deletion (direction check)
- `sdk/typescript/{package.json,tsconfig*.json,scripts/postbuild.mjs,src/generated/contracts.ts}` — the package chain `sdk/react` mirrors; generated typings the kit consumes
- `packages/ui/vitest.config.ts` + `package.json` — React vitest setup reference (kit tests run node-env headless; jsdom only if DOM renders)
- Root `package.json` (workspaces), `scripts/gate.sh` `classify()` (~L131), root `vitest.config.ts` `projects`, `.oxlintrc.json` overrides — the exact wiring edits for the new package
- `internal/extension/testdata/command-fixture-go/` + `sdk/examples/prompt-enhancer/` — fixture + TS-extension layout precedents
- `web/src/systems/os/` view shell/renderers from task_06 — the host runtime integrates here
- `packages/site/content/docs/extensions/develop.mdx` (§ Provide surfaces) — the View-providers section slots beside connectivity/forge

### Dependent Files

- `sdk/typescript/src/generated/contracts.ts` + `sdk/go/contracts/*_gen.go` — capability tables regenerate
- `internal/api/core/transport_parity_integration_test.go` — view-session route rows
- `openapi/compozy.json` + `web/src/generated/compozy-openapi.d.ts` — session route types
- `docs/_memory/glossary.md` — `view.provider` provide-surface entry (P6 co-ship per Impact Audit)
- `.github`/release publishing for `@compozy/extension-react` — explicitly OUT unless the user asks (npm publish wiring is a separate decision; the package ships workspace-internal this cycle)

### Competitor References

- (No `.resources/` subset — the event-loop model is documented in `analysis/05_raycast_vicinae_runtime.md`/`analysis/06_hosts_eventloop.md`; Vicinae is cited by concept, not vendored.)

### Related ADRs

- [ADR-008](adrs/adr-008.md) — the runtime this task implements (channel, event loop, budgets, sessions, protocol additions, `execute_hook` deletion)
- [ADR-009](adrs/adr-009.md) — TS-only programs; `@compozy/extension-react` as a separate workspace package; typing-freshness gates
- [ADR-007](adrs/adr-007.md) — one vocabulary; programs emit the same frames Tier-1 payloads use

### Web/Docs Impact

- `web/`: host runtime for programmable sessions on the view shell (counters, quarantine, bands, circuit-break, effect acks) + session-stream consumer via `createStreamEventSource`; stories for the five band states.
- `packages/site`: `extensions/develop.mdx` View-providers section; React-kit authoring docs (new page or section, navigation-registered); generated API/CLI/contract references.
- QA impact: flag only (walks in task_12): reset the **extension palette contributions** scenario (task_07's) to `untested` — programmable tier added to the same behavior surface (dedup, no new file).

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: FULL — new provide surface (`view.provider`) + host-API method (`view/patch`) + subprocess protocol additions shared by all provides + TS SDK registrar + new React kit package + TS fixture + scaffold template + docs + glossary; `execute_hook` daemon-request seam deleted (delete target); hooks catalog still untouched (checked).
- Agent manageability: view-session routes carry structured errors on both transports; program failures surface through `compozy extension logs` (existing verb — checked, no new verb needed); catalog/list surfaces already expose program views via task_01 verbs.
- Config lifecycle: none — checked: budgets/caps are named constants per Assumptions (deliberately not config); `default_view_timeout_ms` is protocol-negotiated, not a `config.toml` key.

## Deliverables

- `view.provider` capability + session runtime + `view/patch` fan-out live end-to-end against the TS fixture
- Subprocess protocol additions landed for both peers without regressing existing provides (conformance green)
- `sdk/typescript` view-provider surface + `@compozy/extension-react` as a fully wired workspace package (build/typecheck/test/lint/gate lanes)
- Program-specific validation + TS fixture + precise `execute_hook` seam deletion
- Docs/glossary/skills co-ship
- Visual Contract evidence bundles VC-01..05 **(REQUIRED)**
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

Cases assigned from `_tests.md` — read each ID's full definition there before writing tests.

- [x] UT-161, UT-162, UT-163, UT-164, UT-165, UT-166, UT-167, UT-168 — gate parity, session registry ownership/teardown/generation-bump, event counters + `complete:true`, quarantine, degradation state machine with live shell, push fencing, program validation matrix, band budgets with injected clock
- [x] UT-173 — event admission: supersession cancel + late-frame discard + causal-generation rejection of stale pushes + action-event caps + post-close disposal
- [x] UT-169, UT-170, UT-171, UT-172 — React kit: NotesBrowser → valid frame, handler-id stability across re-renders, one patch per tick + dedupe + round-trip property, starvation warning + AbortSignal propagation
- [x] IT-026 — full loop: open (identity-bound) → events → session-scoped patches → at-most-once effects on resync → close → no orphans → foreign-token rejection
- [x] IT-027 — crash + supervisor restart: invalidation, stale-session rejection, fresh reopen
- [x] IT-028 — slow program: degraded marker, circuit-break, per-view isolation
- [x] IT-029 — two clients: independent sessions, no cross-stream frames
- [x] IT-030 — validate matrix: Go `program:true` exact message; TS missing `view/close` describe-coverage
- [ ] E2E-031 — instant typing + chips + push/pop + form submit + destructive confirm on the fixture — walk remains in task_12
- [ ] E2E-032 — bands walk: busy → degraded → circuit-break with shell + sibling views live; crash variant — walk remains in task_12
- [ ] E2E-033 — dev edit → "view reloaded" → next open runs new code — walk remains in task_12

## Success Criteria

- Every assigned test case implemented and passing
- A canceled/superseded handler can never overwrite newer state — even via `InReplyTo=0` pushes (UT-173 pins SI-19)
- A reconnect can never repeat a clipboard write, URL/app open, toast, or file prompt (IT-026 pins SI-21)
- A program crash or stall never takes down the palette, another view, or the daemon (IT-027/IT-028/E2E-032 pin SI-16)
- `sdk/react` passes every repo gate (`make gate` classifies it; oxlint/typecheck/test/build lanes green; 500-line cap respected)
- `rg execute_hook` finds only the live hook-dispatch path — the daemon-request seam is gone
- `make gate` green including conformance + codegen-check
