# Test Specification: Herdr Parity (Session Attention · Orchestration DX · Shortcuts v2)

Canonical test contract for the herdr-parity program. Companion to `_spec.md`.
Derived from `_user_stories.md` (behavior), `_spec.md` Part II (components), `_dx.md` (CLI/API journeys), and `_uiux.md` (browser journeys). Visual language for S1–S14 is the locked OpenDesign set at `docs/design/opendesign/herdr-parity/` — proved by `eng-ui-screenshot` evidence bundles on tasks 03/05/06, not by snapshot/CSS/static tests.

## Strategy

- **Go**: `t.Run("Should …")` subtests, `t.Parallel` default, table-driven, `-race`/`CGO_ENABLED=1`; fakes only at I/O boundaries (store, clock, broadcaster, audio/platform adapters). Integration under `+integration` tag, co-located. Migration coverage extends the canonical fresh/reopen/ahead/integrity/equivalence suites — never new standalone schema tests.
- **Web**: Vitest for pure logic (attention model, notifier, coalescer, view stack, chord parsing, preset diff) with fake timers; Playwright (`make test-e2e-web`) against the daemon-served SPA with MSW/acpmock fixtures updated for the new badges and `session_attention_changed`.
- **Runtime E2E**: Go harness + `acpmock` (`make test-e2e-runtime`) driving clarify/permission/done flows; CLI journeys use the `_dx.md` transcripts verbatim (flags, outputs, exit codes).
- **Visual contract**: named boards under `docs/design/opendesign/herdr-parity/` are normative for visual language (layout, anatomy, tokens, visible states). Evidence is the `eng-ui-screenshot` bundle owned by tasks 03/05/06 — not snapshot/CSS/static tests. An implementation-only screenshot is not parity evidence.
- **Execution**: `make test` / `make test-integration` / `make test-e2e-runtime` / `make test-e2e-web`; `make gate` per phase, `make gate-full` at program close. Contract drift is owned by `make codegen-check`; route-inventory parity by the existing httpapi/udsapi handler tests (extended, not duplicated).

## Coverage Matrix

| Source | Behavior | Unit | Integration | E2E |
| --- | --- | --- | --- | --- |
| US-001 | Distinguish question vs permission | UT-001, UT-002, UT-007 | IT-004 | E2E-001, E2E-002 |
| US-001.EC-1 | Simultaneous question+permission precedence | UT-002, UT-003 | — | — |
| US-001.EC-2 | Pending states survive restart, stay actionable | — | IT-003, IT-004, IT-029 | — |
| US-001.EC-3 | Answered elsewhere clears everywhere | — | IT-005 | E2E-001 |
| US-001.EC-4 | Withdrawn/timed-out question clears | — | IT-022 | — |
| US-001.EC-5 | Non-reporting agent renders unknown | UT-007 | — | E2E-015 |
| US-002 | done = finished while unseen | UT-004 | IT-009 | E2E-003 |
| US-002.EC-1 | Passive reads never mark seen | UT-025 | IT-009 | E2E-003 |
| US-002.EC-2 | done/seen survive restart | — | IT-023 | — |
| US-002.EC-3 | One client focuses → clears for all | UT-081 | — | E2E-009 |
| US-002.EC-4 | done yields to running | UT-005 | — | — |
| US-002.EC-5 | Lifecycle outranks done | UT-006 | — | — |
| US-002.EC-6 | Bulk done stays responsive | UT-052 | — | E2E-009 |
| US-003 | Failures join needs-you | UT-008, UT-051 | — | E2E-009, E2E-010 |
| US-003.EC-1 | Failed while focused → no toast | UT-057 | — | — |
| US-003.EC-2 | Failure flap deduped | UT-055 | — | — |
| US-003.EC-3 | Muted workspace: rows yes, alerts no | UT-034 | — | E2E-009 |
| US-004 | CLI attention visibility + filters | — | IT-018 | E2E-001 |
| US-004.EC-1 | Empty filter result exits 0 | — | IT-018 | — |
| US-004.EC-2 | Invalid state name → vocabulary error | — | IT-018 | — |
| US-004.EC-3 | Token stability is contract-owned | — | — | — (owned by `make codegen-check` + route parity) |
| US-005 | Bell sections + jump | UT-052 | — | E2E-009 |
| US-005.EC-1 | Quiet/empty bell | — | — | E2E-009 |
| US-005.EC-2 | Stale row activation degrades honestly | — | — | E2E-009 |
| US-005.EC-3 | 100+ rows ordered, exact counts | UT-052, UT-080 | — | E2E-009 |
| US-005.EC-4 | Second client updates live | — | — | E2E-009 |
| US-006 | Title carries needs-you count | UT-054 | — | E2E-012 |
| US-006.EC-1 | Title uses cross-workspace total | UT-054 | — | — |
| US-006.EC-2 | Count survives route changes | — | — | E2E-012 |
| US-007 | Attention-first sidebar sort | UT-066 | — | E2E-015 |
| US-007.EC-1 | Stable ties, no jitter | UT-066, UT-080 | — | — |
| US-007.EC-2 | Transition keeps keyboard selection | — | — | E2E-015 |
| US-008 | Needs-you notifications | UT-055, UT-057 | — | E2E-010 |
| US-008.EC-1 | 5 simultaneous → 5 distinct toasts | UT-055 | — | E2E-010 |
| US-008.EC-2 | Same-session flap deduped | UT-055 | — | — |
| US-008.EC-3 | No replay storm after restart | UT-067 | IT-006 | — |
| US-008.EC-4 | Resolved-before-click lands current | — | — | E2E-010 |
| US-009 | Coalesced completions | UT-056 | — | E2E-011 |
| US-009.EC-1 | Needs-you never grouped away | UT-058 | — | — |
| US-009.EC-2 | Window never self-extends | UT-056 | — | — |
| US-009.EC-3 | Group of one/zero collapse | UT-056 | — | — |
| US-010 | Sound on delivery | UT-068 | — | E2E-013 |
| US-010.EC-1 | One sound per batch | UT-068 | — | — |
| US-010.EC-2 | Audio failure silent, visual unaffected | UT-069 | — | — |
| US-011 | Settings round-trip everywhere | UT-047 | IT-011, IT-016 | E2E-013 |
| US-011.EC-1 | First-run defaults | UT-047 | — | — |
| US-011.EC-2 | Concurrent writers last-wins | — | IT-011 | — |
| US-011.EC-3 | Orphan mute pruned | — | IT-024 | — |
| US-012 | System-level escalation opt-in | UT-070 | — | E2E-013, E2E-020 |
| US-012.EC-1 | Permission denied renders truthfully | UT-070 | — | E2E-013 |
| US-012.EC-2 | Unsupported channel absent | UT-070 | — | — |
| US-012.EC-3 | Click after resolve lands current | — | — | E2E-020 |
| US-013 | Agent notify | UT-030–UT-034 | IT-010 | E2E-007 |
| US-013.EC-1 | Caps validation | UT-031, UT-033 | IT-010 | — |
| US-013.EC-2 | Rate limit observable | UT-032 | IT-010 | E2E-007 |
| US-013.EC-3 | Muted outcome truthful | UT-034 | — | — |
| US-013.EC-4 | No client connected → no-client | — | IT-010 | — |
| US-014 | Cross-workspace bell + jump | UT-051 | — | E2E-009, E2E-014 |
| US-014.EC-1 | Removed workspace degrades | — | — | E2E-014 |
| US-014.EC-2 | Jump opens missing window | — | — | E2E-009 |
| US-014.EC-3 | Many workspaces scannable | — | — | E2E-014 |
| US-014.EC-4 | Inaccessible workspace rows vanish | — | — | — (no operator-permission model; removed-workspace path covers) |
| US-015 | Per-workspace mute | UT-034, UT-071 | IT-011 | E2E-013 |
| US-015.EC-1 | Mute drops pending coalesced delivery only | UT-071 | — | — |
| US-015.EC-2 | Unmute never replays | UT-067 | — | — |
| US-016 | CLI wait with predicates/timeout | UT-012–UT-023 | IT-007, IT-008 | E2E-004 |
| US-016.EC-1 | Session gone mid-wait | UT-021 | IT-008 | E2E-004 |
| US-016.EC-2 | Transport drop → deterministic exit 69 | — | IT-025 | — |
| US-016.EC-3 | Invalid token names vocabulary | UT-016 | — | E2E-004 |
| US-016.EC-4 | Concurrent waits both fire | UT-019 | — | — |
| US-016.EC-5 | Fast transit still fires (edge-driven) | UT-013, UT-077 | — | — |
| US-016.EC-6 | Zero/negative timeout rejected; unbounded explicit | UT-023 | — | — |
| US-016.EC-7 | Unbounded is gapless across continuations | UT-084 | — | — |
| US-017 | Native wait tool | — | IT-017, IT-020 | E2E-008 |
| US-017.EC-1 | Cross-workspace denial | — | IT-017 | — |
| US-017.EC-2 | Wait on self denied | — | IT-017 | — |
| US-017.EC-3 | Turn cancel aborts wait cleanly | UT-022 | — | — |
| US-017.EC-4 | Over-cap timeout rejected | UT-023 | IT-017 | — |
| US-018 | Native governed spawn | — | IT-017, IT-019 | E2E-006 |
| US-018.EC-1 | Caps exceeded → named error | — | IT-017 | — |
| US-018.EC-2 | Permission widening rejected with atoms | — | IT-017 | — |
| US-018.EC-3 | Spawn during parent shutdown rejected | — | IT-013 | — |
| US-018.EC-4 | Duplicate spawns are distinct children | — | IT-019 | — |
| US-019 | Wake-on-settle | UT-027–UT-029 | IT-012, IT-013 | E2E-006 |
| US-019.EC-1 | Busy parent → queued, never dropped | — | IT-012 | — |
| US-019.EC-2 | Parent stopped → suppression recorded | UT-029 | IT-012 | — |
| US-019.EC-3 | Flap → dedup per cause-episode | UT-028 | — | — |
| US-019.EC-4 | Reaper/wake race coherent | — | IT-013 | — |
| US-020 | Native stop | — | IT-017 | — |
| US-020.EC-1 | Self-stop denied | — | IT-017 | — |
| US-020.EC-2 | Stop mid-prompt uses the one cancel path | — | IT-026 | — |
| US-020.EC-3 | Cross-workspace denial | — | IT-017 | — |
| US-021 | Native approve, no self-approval | — | IT-017, IT-027 | E2E-006 |
| US-021.EC-1 | Raced → already-resolved | — | IT-027 | — |
| US-021 + US-022 (post-restart) | Orphaned interactions resolvable + discoverable | — | IT-029, IT-032 | — |
| US-021.EC-4 | Queue-full resolution is retryable, loses nothing | — | IT-029 | — |
| US-021.EC-2 | Nothing pending outcome | — | IT-027 | — |
| US-021.EC-3 | Deny reason reaches transcript | — | IT-027 | — |
| US-022 | Native clarify answer | — | IT-017, IT-027 | E2E-006 |
| US-022.EC-1 | Already answered → winner identified | — | IT-027 | — |
| US-022.EC-2 | Unknown request id → not-found | — | IT-017 | — |
| US-022.EC-3 | Oversized answer validated | — | IT-017 | — |
| US-023 | prompt-cancel verb + native tool | — | IT-028, IT-031 | E2E-005 |
| US-023.EC-1 | Cancel twice idempotent | — | — | E2E-005 |
| US-023.EC-2 | Race with natural completion coherent | — | IT-028 | — |
| US-023.EC-3 | Cancel during provider shutdown deterministic | — | IT-026 | — |
| US-024 | Chord arrays | UT-035, UT-036, UT-039, UT-046 | IT-015 | E2E-017 |
| US-024.EC-1 | In-array duplicate handled | UT-040 | — | — |
| US-024.EC-2 | Atomic reject names bad entry | UT-039 | IT-015 | — |
| US-024.EC-3 | Scalar form = 1-element array | UT-035 | — | — |
| US-025 | Range chords | UT-037, UT-041 | — | E2E-017 |
| US-025.EC-1 | Range on non-indexed rejected | UT-038 | — | — |
| US-025.EC-2 | Partial range honest | UT-041 | — | — |
| US-025.EC-3 | Digit-level collision named | UT-042 | — | — |
| US-026 | Terminal preset | UT-063, UT-064 | — | E2E-018 |
| US-026.EC-1 | Idempotent apply; exact revert | UT-064 | — | E2E-018 |
| US-026.EC-2 | Collision preview names actions | UT-063 | — | — |
| US-026.EC-3 | Hazard chords flagged | UT-063 | — | — |
| US-027 | Navigation actions | UT-065, UT-072, UT-073, UT-074 | — | E2E-016 |
| US-027.EC-1 | Zero-attention jump = calm no-op | UT-074 | — | E2E-016 |
| US-027.EC-2 | Cycle with 0/1 sessions | UT-065 | — | — |
| US-027.EC-3 | Missing desktop number no-op | UT-073 | — | — |
| US-027.EC-4 | Editable focus rules respected | — | — | E2E-016 |
| US-027.EC-5 | Attention resolved between press and focus | UT-074 | — | — |
| US-028 | Cheatsheet + full rebindability | UT-075 | IT-015 | E2E-016, E2E-017 |
| US-028.EC-1 | `?` in editable types "?" | — | — | E2E-016 |
| US-028.EC-2 | One canonical surfacing for arrays | UT-075 | — | — |
| US-028.EC-3 | Fresh after rebind without reload | — | — | E2E-017 |
| US-029 | Nested palette views | UT-059, UT-060 | — | E2E-019 |
| US-029.EC-1 | 3+ levels breadcrumb + exact pop order | UT-059 | — | E2E-019 |
| US-029.EC-2 | View empty state, no bleed | UT-060 | — | — |
| US-029.EC-3 | Reopen at root | UT-059 | — | E2E-019 |
| US-029.EC-4 | Live refresh keeps selection | UT-076 | — | — |
| US-030 | Sessions view | UT-061 | — | E2E-016 |
| US-030.EC-1 | Zero-match chip, one-key clear | UT-061 | — | — |
| US-030.EC-2 | List mutation keeps/moves selection | UT-076 | — | — |
| US-030.EC-3 | Hundreds of sessions responsive | — | — | E2E-016 (scale fixture) |
| US-030.EC-4 | Closed window restored on activate | — | — | E2E-016 |
| US-031 | Show all in every session list | — | IT-018 | E2E-014 |
| US-031.EC-1 | Per-group load error isolated | — | — | E2E-014 |
| US-031.EC-2 | Scale across workspaces | — | — | E2E-014 (fixture) |
| US-031.EC-3 | Removed workspace group disappears | — | — | E2E-014 |
| US-031.EC-4 | New workspace joins live | — | — | E2E-014 |
| Badge derivation (Part II: Core Interfaces) | Precedence + classes | UT-001–UT-011 | IT-002–IT-004 | — |
| Wait service | Predicates, pinning, caps, resume, overflow | UT-012–UT-023, UT-077, UT-084, UT-085 | IT-007, IT-008, IT-020 | E2E-004, E2E-008 |
| Seen path | Sole mutation path, monotonic | UT-024–UT-026 | IT-009, IT-023 | E2E-003 |
| Spawn wake bridge | Dedup, suppression, detach, redaction | UT-027–UT-029, UT-079 | IT-012, IT-013 | E2E-006 |
| Pending interactions (canonical records) | Restart recovery, orphan resolution, redaction, discovery | UT-082 | IT-002, IT-029, IT-032 | E2E-002 |
| Presence lease | Per-client focus truth for `done` | UT-004, UT-005, UT-081 | IT-009 | E2E-003 |
| Attention summary | Exact cross-workspace counts | — | IT-033 | E2E-012 |
| Hook `session.attention.changed` contract | Post-commit, cloned, fail-open | UT-086 | — | — |
| Operator-scope authorization | Agent denial on presence / cross-workspace | — | IT-030 | — |
| Attention ordering (`attention_changed_at`) | Durable sort key incl. seen-clears | UT-080 | — | — |
| Notify service | Sanitize, caps, rate limit, transport | UT-030–UT-034, UT-083 | IT-010 | E2E-007 |
| Shortcuts grammar v2 + defaults (ADR-006) | Arrays/ranges/conflicts/effective map | UT-035–UT-046 | IT-015, IT-016 | E2E-017 |
| `[attention]` config | Defaults, validation, overlay | UT-047–UT-049 | IT-011, IT-016, IT-024 | — |
| Web attention pipeline | Tone map, model, notifier, title, SSE consumer | UT-050–UT-058, UT-066–UT-071, UT-078 | — | E2E-009–E2E-015 |
| Palette view stack + Sessions view | Stack, filters, selection | UT-059–UT-062, UT-076 | — | E2E-016, E2E-019 |
| Preset | Diff, apply, revert | UT-063, UT-064 | — | E2E-018 |
| Sessions schema migration | 4 columns, append-only | — | IT-001 | — |
| Attention SSE + wake filter | Event shape, ordering | UT-010, UT-011 | IT-005, IT-006 | E2E-001 |
| Routes (wait/presence/notify/settings) | Parity + shapes + authorization | — | IT-007–IT-011, IT-014, IT-030 | — |
| Native tools chain (7 tools) | Scoping, denials, results | — | IT-017, IT-031 | E2E-006, E2E-008 |

## Unit Tests

### Badge derivation & attention classes (Part II: Core Interfaces)

- **UT-001** (happy): `CanonicalBadge` — `PendingClarify=true`, active state, no auth → `waiting-for-input`.
- **UT-002** (state): `PendingAuth=true ∧ PendingClarify=true` → `waiting-for-auth` (precedence rule 3).
- **UT-003** (state): same inputs with auth resolved (`PendingAuth=false`) → `waiting-for-input` — never `idle`/`running` between.
- **UT-004** (happy): idle-eligible ∧ `Unseen=true` → `done`.
- **UT-005** (state): `ActivePrompt=true ∧ Unseen=true` → `running` (done yields to a new turn).
- **UT-006** (state): terminal state ∧ `Unseen=true` → `stopped`/`failed` (lifecycle outranks attention).
- **UT-007** (boundary): table-driven sweep of the full 10-token precedence order — every adjacent pair beats the next; zero-value inputs → `unknown`.
- **UT-008** (happy): `ClassForBadge` — `waiting-for-input|waiting-for-auth|failed` → `needs-you`; `done` → `finished`; all others → `none`.
- **UT-009** (error): `ClassForBadge` on an unrecognized badge string → `none`, never a panic.
- **UT-010** (state): transition publisher — recompute yielding the same badge publishes nothing; a changed badge publishes exactly one `AttentionEvent`.
- **UT-011** (happy): published event carries `{from, to, class(to), at == attention_changed_at, workspace_id}` — no transcript sequence anywhere in the payload.

### Wait service (`WaitForBadge`)

- **UT-012** (happy): target already in `until` set → immediate `state-reached` with that badge, `Waited≈0`.
- **UT-013** (happy): transition into `waiting-for-input` while waiting → fires even when the state is left within the same tick (edge-driven, not sampled).
- **UT-014** (happy): predicate `[idle]` satisfied by a `done` transition (`done ⊨ idle`).
- **UT-015** (boundary): empty `Until` → settled set `{waiting-for-input, waiting-for-auth, idle, stopped, failed}` applied.
- **UT-016** (error): unknown badge token in `Until` → validation error listing the exact vocabulary; nothing registered.
- **UT-017** (boundary): timeout elapses first → `{Outcome: timeout, Waited == timeout}` (fake clock).
- **UT-018** (state): pinned epoch ≠ session's epoch at fire time → `session-gone`, never `state-reached`.
- **UT-019** (concurrency): two concurrent waits on one session → both fire on the same transition; registry empty afterward.
- **UT-020** (boundary): 33rd concurrent wait on one session → deterministic cap rejection naming the cap.
- **UT-021** (state): session deleted mid-wait → `session-gone`; registration removed.
- **UT-022** (state): caller context canceled → `{Outcome: canceled}`; no leaked watcher (registry empty).
- **UT-023** (error): `Timeout ≤ 0` or `> WaitTimeoutMax` → validation error; the service accepts no unbounded mode — unbounded lives only in the CLI resume loop (UT-084).

### Seen path (`MarkSessionSeen`)

- **UT-024** (happy): sets `last_seen_revision := attention_revision` and `last_seen_at`; recomputed badge drops `done` → `idle`.
- **UT-025** (state): passive reads (status/list/info) never change seen columns.
- **UT-026** (idempotency): repeated `MarkSessionSeen` calls with no new settle → columns unchanged; revisions never regress (monotonic).

### Spawn wake domain

- **UT-027** (happy): `SpawnWakeEvent.Normalize/Validate` — reasons `stopped|failed|needs_attention` valid; empty child id invalid.
- **UT-028** (state): same `WakeEventID` dispatched twice → second is a dedup no-op; a new cause-episode (new id) fires.
- **UT-029** (error): suppression paths (`wake_creator_disabled`, `self_wake`, `session_not_live`, `delivery_failed`) each record their reason and return without error.

### Notify service

- **UT-030** (happy): valid title/body → sanitized publish; outcome from delivery state.
- **UT-031** (error): title of 81 chars post-sanitize → `invalid_input` naming the 80 limit; nothing published.
- **UT-032** (boundary): second notify within 1s from same session → `rate-limited` with `retry_after_ms`; first unaffected.
- **UT-033** (error): empty/whitespace title → validation error.
- **UT-034** (state): sender's workspace muted → `muted-workspace`; nothing delivered, nothing queued.

### Shortcuts grammar v2 + daemon defaults (ADR-006)

- **UT-035** (happy): scalar `"meta+KeyW"` parses as `ShortcutBinding{"meta+KeyW"}`.
- **UT-036** (happy): array input canonicalizes each chord with modifier order `meta+control+alt+shift`.
- **UT-037** (happy): `"control+Digit1..9"` on `desktop.switch` expands to 9 indexed bindings.
- **UT-038** (error): range string on `window.close` → rejected naming the range-capable families.
- **UT-039** (error): chord present in two actions (one inside an array) → rejection names both actions and the chord.
- **UT-040** (error): duplicate chord within one action's array → deterministic rejection naming the entry.
- **UT-041** (boundary): partial range `1..4` expands 4 bindings; settings/effective map reflect exactly 4.
- **UT-042** (error): single-index override colliding with one expanded range digit → diagnostic names the exact digit.
- **UT-043** (happy): `DefaultKeymap()` contains every v2 action (incl. migrated `palette.open`, `session.new`, `scope.global.toggle`, `window.nav.back`); `EffectiveKeymap` merges override-wins.
- **UT-044** (error): unknown action id in overrides → `ErrInvalidCommand`; config validation fails.
- **UT-045** (error): override chord colliding with another action's default (shadowed) → detected server-side, both parties named.
- **UT-046** (happy): empty binding `[]`/`""` disables the action's shortcut; effective map shows it unbound.

### `[attention]` config

- **UT-047** (happy): `DefaultAttentionConfig()` = toasts true, sound true, system false, muted empty; `Validate()` passes.
- **UT-048** (error): malformed `muted_workspaces` entry → `ValidationError{Path:"attention.muted_workspaces"}`.
- **UT-049** (happy): overlay applies pointer-per-field; unset fields keep prior values.

### Web: attention model, notifier, title (Vitest)

- **UT-050** (happy): exported badge→tone/glyph dictionary is exhaustive over all 10 badges (`satisfies` check) — the only tone source (S1/S3/S13/S14).
- **UT-051** (happy): needs-you predicate matches `waiting-for-input|waiting-for-auth|failed`, nothing else.
- **UT-052** (happy): `deriveAttentionRows` — Needs-you section before Finished; within sections newest transition first; counts equal section sizes.
- **UT-053** (state): stale source contributes zero counts and zero rows.
- **UT-054** (happy): title hook — 3 needs-you across 2 workspaces → cross-workspace total in title; 0 → clean title.
- **UT-055** (happy): notifier — 5 sessions entering needs-you in one tick → 5 deliveries; same session flapping within 5s → 1 (fake timers).
- **UT-056** (happy): coalescer — 3 completions within 5s → one "3 sessions finished"; 1 → single form; 0 (all seen before fire) → nothing; window never self-extends.
- **UT-057** (state): transition for the focused session with document visible → no toast/sound; counts still update.
- **UT-058** (state): a needs-you and a completion in the same window → needs-you delivered individually, completion coalesced separately.
- **UT-066** (happy): attention-first comparator — class order needs-you → finished → working → rest; ties by last transition; stable across re-sorts.
- **UT-067** (state): notifier consumes only live-stream edges — replayed/stale-generation events fire nothing (no storm after reconnect/unmute).
- **UT-068** (happy): one sound per delivery batch (5-toast burst → 1 sound; coalesced group → 1).
- **UT-069** (error): audio play rejection swallowed; visual delivery unaffected; no user-facing error.
- **UT-070** (state): system-channel capability model — unsupported → affordance absent; permission denied → unavailable-with-reason; granted → armed.
- **UT-071** (state): muting a workspace mid-coalescing drops its pending completions from delivery, not from bell rows.

### Web: palette view stack, Sessions view, preset

- **UT-059** (happy): push view → breadcrumb path; backspace with empty query pops one; with text edits text; dismiss closes all; reopen starts at root.
- **UT-060** (state): active view renders only its results — root entries never bleed in; view empty state names the filter.
- **UT-061** (happy): Sessions view chips filter by class; zero-match shows empty state; one keystroke clears the chip.
- **UT-062** (happy): TS chord parser accepts scalar/array/range forms and mirrors Go canonicalization (fixture parity with UT-036/037).
- **UT-063** (happy): preset diff lists old→new per action, flags collisions with custom bindings and layout-hazard chords.
- **UT-064** (idempotency): apply twice = once; revert restores the exact pre-preset overrides map.
- **UT-072** (happy): `window.focus.last` picks `focusOrder[1]` (MRU); repeat toggles back.
- **UT-073** (boundary): `desktop.switch.N` beyond the desktop count → calm no-op; comparator matches `switchDesktopDirection` ordering.
- **UT-074** (state): jump-to-attention with zero attention → no-op + hint; target resolving between press and focus → next-best or no-op.
- **UT-075** (happy): cheatsheet rows derive from the effective keymap (defaults + overrides + preset); migrated shell actions listed once; arrays show primary + alternates; surface-local reference section separated.
- **UT-076** (state): live data refresh preserves selection when the selected item still matches; else moves to nearest neighbor.

### Session cycling order

- **UT-065** (state): cycle with 0/1 sessions no-ops; order snapshot frozen during a burst window so consecutive cycles traverse a stable list; wraps at ends.

### Round-1 additions (registration race, SSE consumer, wake redaction, ordering)

- **UT-077** (concurrency): wait registration race — a transition that enters and leaves a target state between subscriber registration and the initial badge snapshot is delivered from the subscription buffer; the wait fires exactly once, never lost, never doubled.
- **UT-078** (happy): the web catalog-stream consumer registers `addEventListener("session_attention_changed")`, routes each payload to the attention notifier exactly once, and removes the listener on teardown/generation supersede (named-SSE invariant).
- **UT-079** (error): `SpawnWakeEvent.Detail` sanitization — secret-shaped content is redacted and the 240-character bound enforced before the synthetic prompt, log line, hook payload, or stream exposure; oversized input truncates deterministically.
- **UT-080** (state): every attention transition — including the event-less seen-clear `done → idle` — bumps `attention_revision` and updates `attention_changed_at`; ordering comparators (bell sections, attention-first sort, Sessions view) order by the timestamp with session-id tie-break, stable across recomputes and restarts; `Unseen` derives from `last_settled_revision > last_seen_revision`, never from transcript sequences.

### Round-2 additions (leases, redaction, notify transport, wait resume, hooks)

- **UT-081** (concurrency): per-client presence leases — client A focused (live lease) and client B hidden: B releasing its own lease never clears A's; a settle racing a renew resolves atomically under the lease-map lock (either seen-under-lease or done, never both); renew/release with a foreign or unknown `lease_id` → deterministic rejection.
- **UT-082** (error): pending-interaction redaction — `compozy_claim_*` and secret-shaped content planted in title, choices, `payload_json`, and resolution are redacted before persistence and before every read surface; payloads over bounds (200/4096/240) truncate or reject deterministically.
- **UT-083** (happy): the web consumer registers `addEventListener("operator_notification")`, renders the sanitized toast with click-to-jump, applies focus/mute rendering rules, and removes the listener on teardown.
- **UT-084** (concurrency): wait resume — a timeout outcome returns `resume_id`; a transition entering and leaving a target between the timeout response and the resumed request is delivered from the kept-alive buffer (gapless); resuming after the 10s grace → `wait-expired`.
- **UT-085** (state): wait edge-buffer overflow (65th buffered edge) → deterministic `overflow` outcome; session stop delivers the `stopped` edge to registered waiters before registry cleanup (stop-vs-wait ordering).
- **UT-086** (state): hook contract for `session.attention.changed` — dispatched at the owning call site after the canonical commit, payload cloned (mutations don't leak), a throwing hook is isolated and logged (fail-open), and no dispatch path tails an event table.

## Integration Tests

### Schema and durable projection

- **IT-001**: migration suites (fresh/reopen/ahead/integrity/equivalence) cover the `session_pending_interactions` side-table and the new `sessions` columns (`pending_permission_count`, `pending_clarify_count`, `attention_revision`, `last_settled_revision`, `last_seen_revision`, `last_seen_at`, `attention_changed_at`); `atlas.sum` and sqlc output current (`make codegen-check`).
- **IT-002**: canonical `session_pending_interactions` row + `sessions` projection columns + `attention_changed_at` commit in one global-catalog transaction — injected failure rolls back all three together; the transcript event appends after commit, and an injected transcript-append failure never corrupts canonical state (badge stays honest).
- **IT-003**: a permission request inserts its canonical `session_pending_interactions` row and bumps `pending_permission_count`; resolution decrements it; daemon restart → badge `waiting-for-auth` before any live event.
- **IT-004**: restart with a pending clarify → badge `waiting-for-input` derived from the column at boot.
- **IT-021** (withdrawn): contract-codegen drift needs no dedicated case — `make codegen-check` (a stronger gate) owns the invariant; cited as completion evidence instead.
- **IT-022**: clarify timeout/withdrawal clears the pending column and the badge leaves needs-you.
- **IT-023**: restart after unseen finish → `done` persists; after `MarkSessionSeen` → restart shows `idle`.

### Eventing

- **IT-005**: wake filter — clarify, done, and error events each publish a catalog wake; unrelated event types do not.
- **IT-006**: `session_attention_changed` publishes only after the canonical global-catalog commit (never gated on the transcript append); carries `at`; no event when the badge is unchanged; an injected transcript-append failure does not block or reorder the publish.

### Routes (HTTP and UDS — both transports per case)

- **IT-007**: `POST …/wait` with `{until:["waiting-for-input"],timeout_ms:120000}` → 200 `state-reached` when the transition lands mid-poll.
- **IT-008**: `/wait` → 200 `{outcome:"timeout"}` at bound; 410 `session_gone` on delete mid-wait; 422 on missing/oversized `timeout_ms`; 404 unknown session.
- **IT-009**: `POST …/presence {visible:true}` → 204 and renews the lease; a settle under a live lease marks seen (never derives `done`); lease expiry followed by a settle → `done`; `{visible:false}` releases early; passive GETs never advance the marker.
- **IT-010**: `POST /api/agent/notify` — `delivered` with a live operator client subscribed; `no-client` with none; `rate-limited` with `retry_after_ms`; 422 over-cap title; requires agent identity.
- **IT-011**: `GET/PATCH /api/settings/attention` round-trip; live apply (no restart); concurrent PATCH last-write-wins consistently.
- **IT-014**: route-inventory parity tests include `/wait`, `/presence`, `/agent/notify`, `/settings/attention` identically on HTTP and UDS.
- **IT-024**: deleting a workspace prunes its `muted_workspaces` entry; settings GET shows no orphan.
- **IT-028**: `/prompt/cancel` + CLI verb — active prompt → canceled with turn id; idle → `nothing-in-flight`; race with natural completion leaves no phantom-running state.
- **IT-029**: pending-interaction lifecycle across restart — the canonical row, counts, and revisions survive; resolving an orphaned (dead-turn) interaction commits resolution + queued prompt in one transaction, reports `resolved-after-restart`, and clears the badge; with the input queue full the transaction aborts with retryable `queue-full` and the record is untouched; duplicate resolution is idempotent (`already-resolved`); concurrent pending requests of the same kind coexist and resolve independently; a resumed provider re-asks under a fresh provider request id (acpmock).
- **IT-030**: operator-scope authorization — agent identity calling `/presence` → `403 agent_scope_denied` (HTTP and UDS, same shape); agent identity requesting the cross-workspace catalog list → same denial; operator Show-all request succeeds; no catalog/stream path serves cross-workspace data to an agent identity.
- **IT-031**: `compozy__session_prompt_cancel` — active prompt canceled with `turn_id` in the structured result; idle target → `nothing-in-flight`; cross-workspace target → standard denial; routes through the same single cancellation path as the verb and route (events record the acting session).
- **IT-032**: pending-interaction discovery — `GET …/interactions?status=pending` lists both kinds sanitized (HTTP and UDS); `compozy session interactions <id>` renders them; session detail/status payloads embed the same projection (native `session_status` inherits it); post-restart the listed ids resolve successfully through approve/answer.
- **IT-033**: attention summary — `GET /api/sessions/attention-summary` returns exact `needs_you`/`finished`/`by_workspace` counts with 100+ sessions across 3 workspaces (beyond any page size); `compozy session list --summary` matches; agent identity → `403 agent_scope_denied`; row pagination on `GET /api/sessions` orders by `attention_changed_at` with a stable cursor.

### CLI

- **IT-016**: `compozy config set attention.sound false` and `window_manager.shortcuts."window.focus.left" '["control+ArrowLeft","alt+KeyH"]'` classify, validate, apply; invalid array member rejects atomically (exit 78).
- **IT-018**: `session list --attention` returns exactly the needs-you class; `--badge done` exact filter; `--all-workspaces` adds workspace column/field; empty → exit 0; invalid badge name → error naming vocabulary (exit 65).
- **IT-025**: `session wait` with the daemon connection killed mid-wait → deterministic exit 69 with transport error message (no hang).

### Native tools

- **IT-017**: for each of the 7 tools — workspace scoping via `nativeSessionInWorkspace` (cross-workspace → standard denial); self-action denials map to `ErrorCodeDenied` + `ReasonApprovalSelfDenied` (approve/answer) or self-target reason (wait/stop/prompt-cancel); structured results match `_dx.md` shapes; spawn maps 409 caps / 422 widening; over-cap wait timeout rejected; oversized clarify answer rejected; unknown request id → not-found.
- **IT-019**: `notify_creator` defaults true across `POST /api/agent/spawn`, CLI spawn, and `compozy__session_spawn` (default applied at validation, not zero-value); CLI `--no-notify-creator` and API/tool `notify_creator:false` disable the wake for that child; duplicate rapid spawns yield distinct child identities.
- **IT-020**: a blocked `compozy__session_wait` emits activity heartbeats — supervision never flags the waiting session inactive (SD-001).

### Wake bridge

- **IT-012**: child stops/fails/enters needs-you → parent receives one synthetic prompt with child id + reason + wake id; busy parent receives it queued (never dropped); parent-stopped and disabled paths record suppression reasons in the audit.
- **IT-013**: parent stopping while child settles (reaper vs wake race) → exactly one coherent outcome, no deadlock, no zombie delivery; spawn during parent shutdown rejected cleanly.
- **IT-026**: stop-tool during active prompt and cancel during provider shutdown both route through the single idempotent cancellation path; events record the acting session.
- **IT-027**: approve/answer already-resolved race → `already-resolved` with winning decision; deny reason lands in the target session transcript; "nothing pending" outcome when no request exists.

### Settings / keymap

- **IT-015**: window-manager settings PATCH with arrays + ranges validates via `CanonicalShortcutsV2` (blocked and shadowed conflicts rejected naming both parties); GET serves `defaults` + `effective` maps; TS registry carries no chord literals (parity fixture, ADR-006).

## End-to-End Tests

### Runtime journeys (Go + acpmock; CLI transcripts verbatim from `_dx.md`)

- **E2E-001**: acpmock asks a clarify → `session status` shows `waiting-for-input`; catalog wake observed; `session clarify answer … --choice 2` → badge leaves needs-you everywhere; `list --attention` empties.
- **E2E-002**: acpmock requests permission → `waiting-for-auth`; `session approve … --decision allow-once` → clears; restart mid-pending preserves the badge.
- **E2E-003**: turn completes with no live presence lease → `done` in list; establishing presence (focusing the session) → `idle`; CLI status reads in between never clear it.
- **E2E-004**: `session wait … --until waiting-for-input,waiting-for-auth,stopped,failed` returns `STATE waiting-for-input` exit 0; `--until idle --timeout 5s` on a busy session exits 75 with the JSON timeout shape; `--until sleeping` exits 65; deleted target exits 69.
- **E2E-005**: `session prompt-cancel` on an active turn → `CANCELED turn …` exit 0; immediately again → `NOTHING IN FLIGHT` exit 66.
- **E2E-006**: the golden path — spawn (wake on), parent continues, child asks → parent woken with the wake prompt; parent answers via `compozy__session_clarify_answer`; parent `compozy__session_wait` settled-set → child `stopped`; zero polling.
- **E2E-007**: `compozy notify "Deps audit done" --body …` → `OUTCOME delivered` (live client connected) / `OUTCOME no-client` (none); burst of 2 in 1s → second `rate-limited`.
- **E2E-008**: acpmock agent turn invokes `compozy__session_wait` on a sibling → structured `state-reached` result; waiting agent stays supervision-green.

### Browser journeys (Playwright; surfaces per `_uiux.md`)

- **E2E-009**: bell — Needs-you (question + permission + failed) and Finished sections; badge counts needs-you only; cross-workspace row activation switches workspace + focuses (opening the window if missing); done row arrival clears its marker for a second open client; quiet state at zero; stale-row activation degrades honestly; 100-row fixture scrolls with exact counts.
- **E2E-010**: needs-you toast fires for an unfocused session with sound once; none for the focused session; click lands on the session (workspace switch included); toast for an already-resolved session lands without error; 5-at-once renders 5 toasts.
- **E2E-011**: three near-simultaneous completions → one grouped toast → opens the bell's Finished section.
- **E2E-012**: backgrounded tab title shows the needs-you total; updates on changes without focus; survives route navigation; clean at zero.
- **E2E-013**: Settings → Attention — toggle sound/toasts round-trips to config; system channel shows denied-with-reason when permission is refused; per-workspace mute silences toasts/sound while bell rows remain.
- **E2E-014**: tri-state Show all — third state lists all workspaces grouped and labeled; per-group error isolated; removed workspace group disappears; new workspace joins live; activation jumps cross-workspace.
- **E2E-015**: sidebar — new badges render with distinct glyph + accessible label (no color-only assertions); attention-first sort floats needs-you; keyboard selection survives a transition re-sort; unknown renders honestly.
- **E2E-016**: keyboard — ⌘E opens the palette in the Sessions view; chips filter; Enter lands and closes; ⌘B toggles the sidebar; `?` opens the cheatsheet (and types "?" inside an input); ⌘K/⌘N still fire inside inputs post-migration; jump-to-attention lands on the latest needs-you session and no-ops calmly at zero; closed-window session activation restores the window; hundreds-of-sessions fixture stays responsive.
- **E2E-017**: Settings → Shortcuts — Tabs group present and rebindable; recording an array alternate persists and the cheatsheet reflects it without reload; blocked and shadowed conflicts render their diagnostics.
- **E2E-018**: preset — preview shows the exact diff (including the ⌘digits→desktops handover); apply is atomic; revert restores the pre-preset table; re-apply is idempotent.
- **E2E-019**: palette nesting — push Sessions view, breadcrumb path, backspace-on-empty pops, dismiss closes all, reopen at root.
- **E2E-020**: system notifications (desktop lane) — with permission granted and app unfocused, a needs-you transition raises an OS notification; activating focuses app → workspace → session; focused app never double-fires.
