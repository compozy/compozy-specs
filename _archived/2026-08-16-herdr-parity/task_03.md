---
status: completed
title: "Web attention surfaces: unified tones, bell sections, title count, Show all, notifier"
type: frontend
complexity: high
---

# Task 3: Web attention surfaces: unified tones, bell sections, title count, Show all, notifier

## Overview

Delivers every operator-facing attention surface in the shell (P2 web + P3 web): the single exported badge→tone/glyph dictionary (killing the G7 duplicate maps), bell with Needs-you/Finished sections and cross-workspace jump, the `document.title` count, attention-first sidebar sort, the tri-state Show all, the attention notifier (toasts with dedup/coalescing/focus suppression, single sound, opt-in system notifications), presence-lease reporting from the session window, and the Settings → Attention page. Covers `_uiux.md` S1–S8 + S14.

<critical>
- ALWAYS READ `_spec.md` and its catalogs (`_user_stories.md`, `_dx.md`, `_uiux.md` when present, `_tests.md`) before starting
- REFERENCE `_spec.md` Part II for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. One exported badge→tone/glyph dictionary MUST feed S1/S3/S13/S14 (exhaustive `satisfies` over all 10 badges); the local maps in `session-list-row.tsx:6-12` and `session-status-line.tsx:13-22` are delete targets. No state is conveyed by color alone (shape/glyph + accessible label per US-001.AC-4); the locked mapping in `_uiux.md` + `docs/design/opendesign/herdr-parity/DESIGN-NOTES.md` is the authority (`DESIGN.md` grammar, no invented tokens).
2. The bell MUST render Needs-you (sessions + existing task-approval rows) and Finished sections; badge + title count = cross-workspace needs-you total from `GET /api/sessions/attention-summary` — never from row pages (round-2 B-113); rows order by `attention_changed_at`.
3. Row/toast/jump activation MUST converge on one land-on-session behavior: switch workspace when needed (`setActiveWorkspaceId` + workspace-switch barrier), then `routing-coordinator` `userOpen`/`userActivateWindow`; arrival at a `done` session is marked seen via presence.
4. The session window MUST report presence: acquire on focus+visible, renew ~5s, release on blur (`lease_id` protocol per `_dx.md`); passive surfaces never report presence.
5. The notifier MUST consume `session_attention_changed` (edges → toast/sound/system decisions client-side) and `operator_notification` (agent notifications) via named listeners with generation-fenced cleanup; state always refetches on catalog wakes (never patched from attention events); stale streams fire nothing.
6. Delivery rules MUST match Business Rules 15–19: per-session 5s dedup, 5s completion coalescing (group-of-one collapses; needs-you never coalesced), focus suppression, per-workspace mute silences delivery but not counters, one sound per delivery batch (silent on autoplay failure), system notifications opt-in and only while the app is unfocused/hidden with truthful permission states.
7. The sidebar MUST gain the attention-first sort (persisted per operator in the daemon-backed shell-state store, not localStorage) and the tri-state `recent | all | all workspaces` toggle with workspace grouping, per-group error isolation, and live workspace join/leave.
8. Settings → Attention MUST round-trip the `[attention]` section (adapter PATCH full-config per the window-manager exemplar) with truthful system-channel states.
9. The catalog stream store MUST follow the `worktree-catalog-stream-store.ts` shape (generation fencing, WeakMap closer, disabled/connecting/live/stale) — extend the existing session stream store rather than adding a fourth near-identical store.
10. MUST implement from the locked visual contracts in `docs/design/opendesign/herdr-parity/` (`DESIGN-NOTES.md` + the boards named in `## Visual Contract`). Activate `eng-design` + `ui-craft` + `impeccable` and `eng-ui-screenshot` Visual Contract Mode before code. Artboard CSS is a contract, never a stylesheet to import. MUST produce the `eng-ui-screenshot` evidence bundle for every Visual Contract row — implementation-only captures are invalid.
</requirements>

## Visual Contract

Reference artboards: `docs/design/opendesign/herdr-parity/` (iterate, never regenerate). All rows viewport 1440×900, fidelity normative. Authorized differences: `DESIGN-NOTES.md` lossiness rules (runtime truth + `COPY.md` own content/copy; `@compozy/ui` owns component identity; live host chrome owns placement context; S4 title wording and native-notification copy are COPY.md proposals) — record each divergence as an authorized delta in `review.md`.

| ID | Reference artifact + state | Implementation target + state | Viewport | Fidelity |
| --- | --- | --- | --- | --- |
| VC-01 | `herdr-parity-sidebar.html` — §01 dictionary (all 10 badges) | Storybook `SessionBadgeMark` all-states story | 1440×900 | normative |
| VC-02 | `herdr-parity-sidebar.html` — §02 Sessions window (Recent, attention-first option) | Sessions window populated fixture | 1440×900 | normative |
| VC-03 | `herdr-parity-sidebar.html` — §03 sort menu | Sessions sort popover | 1440×900 | normative |
| VC-04 | `herdr-parity-sidebar.html` — §03 per-group failure (`Couldn't load sessions · Retry`) | Show-all group error fixture | 1440×900 | normative |
| VC-05 | `herdr-parity-sidebar.html` — §03 collapsed group | Show-all collapsed fixture | 1440×900 | normative |
| VC-06 | `herdr-parity-sidebar.html` — §04 precedence (auth outranks input, `+1 question`) | Sidebar row precedence fixture | 1440×900 | normative |
| VC-07 | `herdr-parity-sidebar.html` — §04 done → running hand-off | Sidebar row hand-off fixture | 1440×900 | normative |
| VC-08 | `herdr-parity-sidebar.html` — §05 status line `waiting-for-input` | Session window status line question | 1440×900 | normative |
| VC-09 | `herdr-parity-sidebar.html` — §05 status line `done` | Session window status line finished-unseen | 1440×900 | normative |
| VC-10 | `herdr-parity-sidebar.html` — §05 status line `running` | Session window status line running | 1440×900 | normative |
| VC-11 | `herdr-parity-bell.html` — §01 both sections, cross-workspace | Bell popover populated | 1440×900 | normative |
| VC-12 | `herdr-parity-bell.html` — §02 needs-you-only | Bell needs-you-only | 1440×900 | normative |
| VC-13 | `herdr-parity-bell.html` — §02 finished-only | Bell finished-only | 1440×900 | normative |
| VC-14 | `herdr-parity-bell.html` — §02 quiet (`All quiet`) | Bell empty | 1440×900 | normative |
| VC-15 | `herdr-parity-bell.html` — §02 disconnected | Bell `os-bell-disconnected` | 1440×900 | normative |
| VC-16 | `herdr-parity-bell.html` — §02 muted workspace row (`bell-off`) | Bell muted-workspace fixture | 1440×900 | normative |
| VC-17 | `herdr-parity-toasts.html` — §01 stack (needs-you + coalesced + agent-sent) | Toast stack fixture | 1440×900 | normative |
| VC-18 | `herdr-parity-toasts.html` — §02 needs-you variants (input / auth / failed) | Toast variant story | 1440×900 | normative |
| VC-19 | `herdr-parity-toasts.html` — §03 4-visible fold (`+N more need you`) | Toast overflow ledge | 1440×900 | normative |
| VC-20 | `herdr-parity-toasts.html` — §03 resolved-before-click landing (quiet info notice) | Session landing after stale toast | 1440×900 | normative |
| VC-21 | `herdr-parity-settings-attention.html` — §01 Settings → Attention (defaults) | Settings Attention page | 1440×900 | normative |
| VC-22 | `herdr-parity-settings-attention.html` — §02 system channel Armed / Denied / Unavailable | Settings system-channel chips | 1440×900 | normative |
| VC-23 | `herdr-parity-settings-attention.html` — §03 mute list empty | Settings mute-list empty | 1440×900 | normative |

Evidence for each row: `.compozy/tasks/herdr-parity/evidence/visual/task_03/<contract-id>/{reference.png,implementation.png,side-by-side.png,diff.png,comparison.json,review.md}`.

## Subtasks

- [x] 3.1 Exported tone/glyph dictionary + S1/S14 adoption; delete the two local maps.
- [x] 3.2 Attention model rewrite: class-based predicates, sections, summary-fed counts, `attention_changed_at` ordering (attention-model.ts stays pure + staleness-first).
- [x] 3.3 Bell sections + workspace labels + jump; quiet/stale/scale states per US-005/US-014 ECs.
- [x] 3.4 `useDocumentTitleBadge` hook (summary-fed, route-change safe).
- [x] 3.5 Presence reporting from the session window (acquire/renew/release lifecycle).
- [x] 3.6 Notifier: named-listener consumption, dedup/coalescing/suppression/mute, sound adapter, system-notification channel with permission flow (browser; desktop plugin + capability if the desktop lane is in reach — otherwise truthful unsupported state).
- [x] 3.7 Sidebar: attention-first sort (persisted) + tri-state Show all with grouping.
- [x] 3.8 Settings → Attention section page + adapter + stories.
- [x] 3.9 MSW fixtures/stories updates; site and official-skill configuration docs updated for daemon-backed shell preferences.
- [x] 3.10 Visual-contract capture bundles for VC-01..VC-23 — completed in task_08; all 23 bundles validate with zero blocking divergences.

## Implementation Details

Reference `_spec.md` Part II (Key Decisions: client-side delivery, seen-clear propagation), `_uiux.md` S1–S8/S14 (states from cited US ACs/ECs), and `docs/design/opendesign/herdr-parity/DESIGN-NOTES.md` (locked visual facts).

### Relevant Files

- `web/src/systems/session/components/session-list/session-list-row.tsx:5-25` (SessionStatusMark + local map), `session-status-line.tsx:13-22` (duplicate map), `session-list.tsx` (`view: "recent" | "all"` toggle → tri-state), `lib/session-hierarchy.ts`.
- `web/src/systems/os/lib/attention-model.ts` (81L pure functions) + `web/src/systems/os/hooks/use-os-attention.ts` (154L; workspace-wide authority invariant, staleness gating) + `attention-bell.tsx` + `os-menubar.tsx:109-115,193-204`.
- `web/src/systems/session/hooks/use-session-catalog-streams.ts:15,72-83` + `web/src/systems/workspace/hooks/worktree-catalog-stream-store.ts` (the store shape to follow).
- `web/src/hooks/use-document-visible.ts` — the `useSyncExternalStore` pattern for the title hook and visibility gating.
- `web/src/lib/user-feedback.ts` + `packages/ui/src/components/sonner.tsx` + `web/src/main.tsx:52` — toast pipeline.
- `web/src/systems/os/lib/routing-coordinator.ts:178,214,246` (`userOpen`/`userActivateWindow`) + `web/src/systems/workspace/hooks/use-active-workspace.ts:17,41` + `os/lib/workspace-switch-barrier.ts` — the jump.
- `web/src/systems/settings/adapters/window-manager-layouts-api.ts:81-90` + `web/src/systems/settings/routes/` — settings section exemplar.
- `web/src/lib/status-tone.ts` — candidate home for the exported dictionary (already hosts `TASK_STATUS_TONE` with `satisfies`).
- `web/src/systems/os/hooks/use-os-sessions-modal.ts:11` + `use-session-sidebar-state.ts:84-95` — daemon-backed shell-state persistence convention.
- `docs/design/opendesign/herdr-parity/DESIGN-NOTES.md` + `herdr-parity-sidebar.html` / `herdr-parity-bell.html` / `herdr-parity-toasts.html` / `herdr-parity-settings-attention.html` — locked visual contracts for S1–S8/S14.

### Dependent Files

- `web/src/systems/os/lib/dock-badges.ts` + `use-desktop-dock.ts:60,74` — dock counts inherit the class predicate.
- `web/src/systems/os/components/os-command-palette-results.tsx` — task_06 consumes the dictionary + model.
- `web/e2e/` Playwright suite + `web/src/storybook/openapi-msw.ts` — new fixtures/events.
- `desktop/src-tauri/` (capabilities + Cargo) — system-notification plugin when the desktop lane ships.

### Related ADRs

- [ADR-001](adrs/adr-001.md) — class rendering rules; [ADR-002](adrs/adr-002.md) — delivery policy; [ADR-005](adrs/adr-005.md) — event consumption contract (edges ephemeral, wakes authoritative).

### Competitor References

- `.resources/herdr/src/ui/status.rs:195,237` + `src/workspace/aggregate.rs:75` — indicator glyph/color pairs and attention-priority aggregation (blocked 4 > done 3 > working 2 > idle 1).
- `.resources/herdr/src/app/actions.rs:111,155` — focused-pane suppression (`active_tab_suppresses_notifications`) and `open_notification_target` jump behavior.
- `.resources/herdr/src/platform/macos.rs:583` — system-notification delivery + activate-on-click pattern.

### Web/Docs Impact

- `web/`: everything above (systems `os`, `session`, `settings`, `workspace`; hooks, components, stores, mocks, stories).
- `packages/site`: added `[shell.sessions]` to the config reference and regenerated the lifecycle matrix; no standalone notification UX page was needed because the implementation introduced no public behavior beyond the approved spec and configuration contract.
- QA impact: added `J-respond-to-agent-attention` and six content-addressed scenarios for bell/jump, toast delivery, title count, channel states, all-workspaces scope, and attention sort. Task 08 walked and passed them and materialized VC-01..VC-23.

### Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: consumes the task_01/02 named session catalog events without defining a parallel event or hook; no extension manifest, bridge SDK, MCP sidecar, or external registry contract changed. The shell preference section is registered through the existing config, lifecycle, settings, and generic config-tool registries.
- Agent manageability: agents can read and change the global preferences with `compozy config get/set shell.sessions.{sort,scope}` and the HTTP/UDS `GET/PATCH /api/settings/shell` section. No new native tool ID is needed because the generic config surface already owns these paths.
- Config lifecycle: added global-only, live `[shell.sessions]` keys: `sort = "last_activity" | "attention"` and `scope = "recent" | "all" | "all-workspaces"`; defaults, validation, overlay, global write policy, active projection, generated contract, CLI, site docs, and official-skill docs ship together. Workspace config rejects this section.

## Completion Notes

The implementation and controller audit are complete. The audit repaired cursor completeness,
cross-workspace activation barriers, event-driven query wakes, source-specific staleness, presence
renewal overlap, full-section write races, application-focus suppression, stale-toast landing, and
the four-visible toast overflow ledge. A full web-unit pass exposed and the controller fixed an
accidental QueryClient dependency in the sessions modal; its 9-test canonical suite then passed.
Focused Go tests with race, strict Go lint, web typecheck/lint, codegen consistency, the other 625 web
test files, and the assigned Vitest suites pass. Playwright execution, visual captures, real-user QA,
and the full monorepo gate remain intentionally deferred to tasks 07–08 and the workstream tail.

Compozy Impact Audit:

- Native tools: no tool IDs, toolsets, schemas, digests, risk flags, or capability gates changed; checked the generic config get/set path, which now accepts the registered global `shell.sessions.*` fields without a new descriptor.
- Extensibility and hooks: named `session_attention_changed` and `operator_notification` consumers use the existing catalog stream; no hook, extension resource, bridge SDK, MCP sidecar, or external registry changed. The new shell section is wired through the existing config/lifecycle/settings registries.
- Workspace data isolation: attention summary and attention-row reads are operator-scoped cross-workspace projections; ordinary lists, details, presence, query keys, cache invalidation, SSE wakes, and window actions retain workspace ownership. Shell preferences are global operator data and workspace writes are rejected. Cursor-complete group queries keep failures and caches keyed per workspace.
- Official Compozy skill: updated `skills/compozy/references/configuration.md` with the public shell preference values, global-only scope, live lifecycle, CLI paths, and HTTP/UDS section; the audit receipt is `evidence/task_03-compozy-skill-audit.md`.

## Deliverables

- All nine surfaces implemented per `_uiux.md` states with `@compozy/ui` reuse (no new primitives) and zero color-only states.
- Notifier obeying every Business Rule 15–21 behavior client-side.
- Stories + MSW fixtures updated; Vitest + Playwright suites green through Turbo from repo root.
- Every Visual Contract row has a durable passing evidence bundle **(REQUIRED)**.
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**.

## Tests

- [x] UT-050..UT-058 — dictionary, model, notifier, title, suppression, coalescing
- [x] UT-066..UT-071 — comparator, replay-safety, sound batching/failure, system-channel states, mute-mid-coalesce
- [x] UT-078 — `session_attention_changed` named-listener registration/routing/cleanup
- [x] UT-083 — `operator_notification` named-listener + toast + click-to-jump
- [x] E2E-009..E2E-015 — authored and passed in task_08's focused and full Web E2E runs
- [ ] E2E-020 — desktop lane not shipped; browser renders the truthful unsupported state; final disposition belongs to task_08

## Success Criteria

- Every assigned test case implemented and passing.
- Every Visual Contract row is `PASS` with zero unresolved blocking divergence.
- Counters always equal the summary projection; `done` never inflates needs-you; a second client's `done` clears on the first client's focus (US-002/US-005/US-006 ACs).
- Zero toasts from stale streams or for the focused session; grouped completions; per-workspace mute honored.
- `make gate` green (web lanes via Turbo); no `@compozy/ui` shadowing lint errors.
