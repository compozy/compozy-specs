# Test Specification: Command Palette — OS-Grade Overhaul

Canonical test contract for the command-palette overhaul. Companion to `_spec.md`. Derived from `_user_stories.md` (behavior), `_spec.md` Part II (components), `_dx.md` (CLI/API journeys), and `_uiux.md` (browser journeys).

## Strategy

- **Go unit**: testify, `t.Run("Should …")` + `t.Parallel`, table-driven, `-race`/`CGO_ENABLED=1`; injected clocks for every decay/ranking case; store faked only at the sqlc boundary. Ranking carries a property test (comparator transitivity) alongside table cases.
- **Web unit**: Vitest + RTL, extending the canonical suites (`use-os-command-palette.test.tsx` lineage, pure-lib suites, `packages/ui` command tests); cmdk async filtering wrapped in `waitFor`.
- **Integration**: `+integration` tag; real globaldb temp store with migrations applied; both transports exercised where the parity suite doesn't already; extension fixture with a `cmd_palette` manifest block.
- **E2E**: web via Playwright (extends `os-shell.spec.ts`); runtime via the Go harness against `acpmock` over UDS; desktop via Playwright `_electron` in `desktop/e2e/`. CLI/API journeys use the `_dx.md` transcripts verbatim.
- Contract co-ship: OpenAPI + generated TS + `native-tool-catalog.json` + E2E mock matchers land with the contract change.

## Coverage Matrix

| Source | Behavior | Unit | Integration | E2E |
| --- | --- | --- | --- | --- |
| US-001 | Full catalog, every action present + bindable | UT-001, UT-002 | IT-001 | E2E-001 |
| US-001.EC-1 | Daemon loss → disabled-with-reason, exempt survive | UT-096, UT-101 | — | E2E-018 |
| US-001.EC-2 | Catalog scale: sections ordered, honest overflow | UT-113 | — | E2E-002 |
| US-001.EC-3 | Feature-disabled command hidden | UT-097 | — | — |
| US-001.EC-4 | Duplicate id rejected at load | UT-004, UT-056 | IT-016 | — |
| US-002 | Fuzzy match + deterministic order | UT-020, UT-021, UT-025 | — | E2E-001 |
| US-002.EC-1 | Regex chars treated literal | UT-026 | — | — |
| US-002.EC-2 | Overlong query → empty + fallback row | UT-111, UT-140 | — | E2E-013 |
| US-002.EC-3 | Async waves merge without reorder/selection theft | UT-114 | — | E2E-003 |
| US-002.EC-4 | Term-must-match-or-drop | UT-022 | — | — |
| US-003 | Entity search across all domains | UT-110, UT-112 | — (web-owned; IT-002 withdrawn) | E2E-003 |
| US-003.EC-1 | Stale entity ⏎ → honest error | UT-106 | — | E2E-003 |
| US-003.EC-2 | Domain at 100× volume capped + exact note | UT-113 | — | — |
| US-003.EC-3 | Domain endpoint error → inline section error | UT-115 | — | E2E-003 |
| US-003.EC-4 | Vault values never render | UT-112 | — | — |
| US-003.EC-5 | Same-name entities show workspace labels | UT-116 | — | E2E-004 |
| US-004 | Settings pages/keys searchable | UT-012 | IT-001 | E2E-005 |
| US-004.EC-1 | Stateless-gated settings result hidden | UT-097 | — | — |
| US-004.EC-2 | Command vs settings name collision → both groups | UT-117 | — | E2E-005 |
| US-005 | Empty-query: pins → recents → curated | UT-030, UT-095 | IT-006 | E2E-006 |
| US-005.EC-1 | Stale pin/recent pruned tolerantly | UT-089 | IT-006 | — |
| US-005.EC-2 | Unavailable recents disabled-not-hidden | UT-098 | — | — |
| US-006 | Ghost autocomplete | UT-118 | — | E2E-001 |
| US-006.EC-1 | Ghost preserves typed casing | UT-119 | — | — |
| US-006.EC-2 | → mid-input moves caret only | UT-119 | — | — |
| US-007 | One scope model, persisted globe | UT-099 | IT-003 | E2E-004 |
| US-007.EC-1 | Deleted workspace rows vanish; stale ⏎ honest | UT-106 | — | — |
| US-007.EC-2 | Single-workspace globe still works | UT-099 | — | — |
| US-008 | Context-aware availability/boost from daemon truth | UT-100, UT-024 | — | E2E-007 |
| US-008.EC-1 | Live context change re-ranks without selection theft | UT-114 | — | E2E-007 |
| US-008.EC-2 | Contradictory context degrades to disabled | UT-100 | — | — |
| US-008.EC-3 | Context unavailable ≠ allow-all | UT-101 | — | — |
| US-009 | Stack semantics across all view kinds | UT-130 | — | E2E-008 |
| US-009.EC-1 | Unknown view id → unavailable frame; pop works | UT-131 | — | E2E-017 |
| US-009.EC-2 | Depth ≥5 breadcrumb contract | UT-132 (existing `palette-view-stack` suite) | — | — |
| US-009.EC-3 | Destination ∧ stack mutual exclusion | UT-107 | — | E2E-015 |
| US-010 | Every list domain gets a curated view | UT-133 | — (web-owned; IT-004 withdrawn) | E2E-009 |
| US-010.EC-1 | Empty-with-filter honest state + 1-key clear | UT-134 | — | E2E-009 |
| US-010.EC-2 | Live refresh keeps selection | UT-114 | — | — |
| US-010.EC-3 | Cold-cache loading ≠ empty | UT-134 | — | — |
| US-010.EC-4 | Vault view names-only | UT-112 | — | — |
| US-011 | Detail accessory pane | UT-135 | — | E2E-010 |
| US-011.EC-1 | Deleted entity clears pane | UT-135 | — | — |
| US-011.EC-2 | Long content scrolls independently | UT-135 | — | — |
| US-011.EC-3 | Hostile markdown sanitized/degraded | UT-136 (sole owner; UT-041 = wire caps only) | — | — |
| US-012 | Form views: typed fields, validation, submit | UT-137 | IT-005 | E2E-011 |
| US-012.EC-1 | Pop discards values; re-push clean | UT-137 | — | — |
| US-012.EC-2 | Downstream failure keeps form + values | UT-138 | — | E2E-011 |
| US-012.EC-3 | Password masked, cleared, never logged | UT-138, UT-092 | — | — |
| US-012.EC-4 | Empty dropdown declared hint; required blocks | UT-137 | — | — |
| US-013 | Grid views: 2D nav + parity | UT-139 | — | E2E-012 |
| US-013.EC-1 | Media failure → placeholder | UT-139 | — | — |
| US-013.EC-2 | Empty grid grammar | UT-139 | — | — |
| US-013.EC-3 | Grid scale cap/virtualized | UT-113 | — | — |
| US-014 | Action panel: filter, sections, primary ↩, meta-actions | UT-125, UT-126 | — | E2E-010 |
| US-014.EC-1 | Vanished row closes panel, no dead fire | UT-127 | — | — |
| US-014.EC-2 | Disabled row panel: meta + reason only | UT-128 | — | — |
| US-014.EC-3 | Capture-phase action shortcut + repeat guard | UT-129 | — | — |
| US-015 | Inline typed arguments | UT-120 | IT-007 | E2E-011 |
| US-015.EC-1 | Esc restores search, values discarded | UT-121 | — | — |
| US-015.EC-2 | Type-invalid paste blocks with message | UT-121 | — | — |
| US-015.EC-3 | Hotkey → opens in argument mode | UT-122 | — | E2E-028 |
| US-015.EC-4 | Password arg never echoed/learned | UT-092 | IT-008 | — |
| US-016 | Declared confirmation, Cancel focused | UT-123 | — | E2E-013 |
| US-016.EC-1 | Repeat-guarded confirm | UT-124 | — | — |
| US-016.EC-2 | Target changed → confirmation invalidates | UT-124 | — | — |
| US-016.EC-3 | Agent path maps to approval, no bypass | UT-010 | IT-009 | E2E-024 |
| US-017 | Honest execution feedback | UT-159 | IT-010 | E2E-014 |
| US-017.EC-1 | Mid-invoke daemon drop → failure reported | UT-160 | IT-010 | — |
| US-017.EC-2 | Non-idempotent single-flight 409 | UT-009 | IT-010 | — |
| US-017.EC-3 | Cross-workspace landing named | UT-159 | — | E2E-004 |
| US-018 | Frecency evolves and decays | UT-027, UT-028 | IT-006 | E2E-006 |
| US-018.EC-1 | Cold start → curated tiers | UT-030 | — | — |
| US-018.EC-2 | Corrupt store → defaults + one report | UT-090 | IT-011 | — |
| US-018.EC-3 | Stale usage ids pruned on read | UT-089 | — | — |
| US-019 | Query learning boosts learned target | UT-029 | IT-006 | — |
| US-019.EC-1 | Learned target gone → pruned | UT-089 | — | — |
| US-019.EC-2 | Conflicting associations: recent wins, deterministic | UT-029 | — | — |
| US-019.EC-3 | Workspace isolation of learning | UT-091 | IT-011 | — |
| US-020 | Pins: pin/unpin, rest-state placement | UT-030 | IT-006 | E2E-006 |
| US-020.EC-1 | Stale pin pruned | UT-089 | — | — |
| US-020.EC-2 | Double-pin idempotent | UT-093 | — | — |
| US-021 | Recents group + reset | UT-030, UT-094 | IT-012 | E2E-006 |
| US-021.EC-1 | Reset converges other tabs on refresh | — | IT-012 | — |
| US-021.EC-2 | Unavailable recents per US-005.EC-2 | UT-098 | — | — |
| US-022 | Rebind any command; conflicts named; reset | UT-070, UT-071, UT-148 | IT-013 | E2E-016 |
| US-022.EC-1 | Surface-local shadow warning extended | UT-072 | — | — |
| US-022.EC-2 | Bare-letter rejection outside exempt contexts | UT-073 | — | — |
| US-022.EC-3 | Unknown-id override tolerated with diagnostic | UT-074 | IT-013 | — |
| US-022.EC-4 | Concurrent binding edits converge | — | IT-013 | — |
| US-023 | Aliases rank first, render "(alias)" | UT-023, UT-149 | IT-014 | E2E-016 |
| US-023.EC-1 | Alias grammar 1–32 no whitespace | UT-081 | IT-014 | — |
| US-023.EC-2 | Alias vs title collision: alias owner first | UT-023 | — | — |
| US-023.EC-3 | Removed command prunes alias | UT-089 | — | — |
| US-024 | Global summon + per-command OS hotkeys | UT-155, UT-156 | — | E2E-027, E2E-028 |
| US-024.EC-1 | macOS accessibility surfaced with deep-link | UT-157 | — | E2E-029 |
| US-024.EC-2 | Shell quit releases; relaunch re-registers/reports | UT-156 | — | E2E-029 |
| US-024.EC-3 | Non-QWERTY limitation surfaced to the user | — (owned by the P8 QA-scenario walk — prose/copy is not unit-testable per Critical Rules) | — | — |
| US-024.EC-4 | Summon over modal focuses without executing through it | — | — | E2E-027 |
| US-025 | Hints + live cheatsheet incl. extension bindings | UT-102, UT-151 | — | E2E-016 |
| US-025.EC-1 | Rebind updates badges live | UT-102 | — | E2E-016 |
| US-025.EC-2 | Multi-chord display rules | UT-151 | — | — |
| US-026 | Fallback row → new session with query prompt | UT-140, UT-141 | IT-015 | E2E-013 |
| US-026.EC-1 | No default agent → picker path | UT-142 | — | E2E-013 |
| US-026.EC-2 | Spawn failure → toast + query preserved | UT-143 | — | — |
| US-026.EC-3 | Nothing pre-sends before ⏎ | UT-141 | IT-015 | — |
| US-026.EC-4 | Repeat ⏎ distinct sessions, repeat-guarded | UT-144 | — | — |
| US-027 | Manifest palette commands: appear/disappear, namespaced | UT-055, UT-057 | IT-016 | E2E-020 |
| US-027.EC-1 | Core/ext id collision rejected with diagnostic | UT-056 | IT-016 | — |
| US-027.EC-2 | Unavailable backing tool → disabled with reason | UT-058 | IT-017 | — |
| US-027.EC-3 | Hostile strings sanitized/length-capped at validate | UT-059 | — | — |
| US-027.EC-4 | Crash-looping ext leaves catalog with health reason | UT-060 | IT-017 | — |
| US-027.EC-5 | Per-workspace instance declarations (dev overlay) | UT-061 | IT-018 | — |
| US-028 | Extension views render with built-in contract | UT-040, UT-042 | IT-019 | E2E-020 |
| US-028.EC-1 | Invalid payload → honest error frame + diagnostics | UT-043 | IT-019 | — |
| US-028.EC-2 | Slow source → loading, timeout, retry; stack never wedges | UT-131 | — | — |
| US-028.EC-3 | Oversized payload capped with exact overflow | UT-044 | — | — |
| US-028.EC-4 | Unknown kind: validate-fail authored; runtime honest frame | UT-045 | — | — |
| US-028.EC-5 | Ext disabled mid-view → unavailable frame named | UT-131 | IT-017 | — |
| US-029 | Default shortcuts bind-only-if-free; dormant visible | UT-075, UT-076 | IT-016 | — |
| US-029.EC-1 | Two exts same chord: enable-order deterministic | UT-076 | — | — |
| US-029.EC-2 | Disable deactivates; user overrides persist dormant | UT-077 | IT-016 | — |
| US-029.EC-3 | Malformed chord fails validate naming grammar | UT-062 | — | — |
| US-030 | Tool policy inheritance (risk/approval/trust) | UT-010 | IT-009 | E2E-024 |
| US-030.EC-1 | Denied approval → failure reason, no partial effect | — | IT-009 | E2E-024 |
| US-030.EC-2 | Untrusted workspace → disabled with trust reason | UT-058 | IT-017 | — |
| US-030.EC-3 | Policy change mid-open disables next render | UT-098 | — | — |
| US-031 | Dev hot reload of contributions | — | IT-018 | E2E-021 |
| US-031.EC-1 | Reload with open view refreshes or pops gracefully | UT-131 | IT-018 | — |
| US-031.EC-2 | Broken reload keeps last-good + surfaces error | UT-063 | IT-018 | — |
| US-032 | Agent enumerates registry structured | UT-011 | IT-001 | E2E-022 |
| US-032.EC-1 | Full-scale listing complete; jsonl streams | — | IT-001 | E2E-022 |
| US-032.EC-2 | Degraded subsystem reported, never silently shorter | UT-005 | IT-017 | — |
| US-033 | Agent invokes by id under policy | UT-008 | IT-009 | E2E-023 |
| US-033.EC-1 | Unknown id 404, no fuzzy invoke | UT-008 | IT-009 | E2E-023 |
| US-033.EC-2 | Schema-invalid args 422 with fields | UT-007 | IT-009 | E2E-023 |
| US-033.EC-3 | Context-unavailable 412 with UI-identical reason | UT-006 | IT-009 | — |
| US-033.EC-4 | Concurrent non-idempotent 409 | UT-009 | IT-010 | — |
| US-034 | Agent manages bindings/aliases/personalization | — | IT-013, IT-014, IT-012 | E2E-025 |
| US-034.EC-1 | API grammar errors match UI substance | UT-081 | IT-014 | — |
| US-034.EC-2 | Conflict write 409 naming owner; overwrite flag | UT-071 | IT-013 | — |
| US-035 | Menubar derives from registry | UT-145, UT-146 | — | E2E-019 |
| US-035.EC-1 | Unavailable menu item disabled-with-reason, stable | UT-146 | — | — |
| US-035.EC-2 | Ext disable removes curated-in items cleanly | UT-147 | — | — |
| US-036 | Destination mode registry-driven | UT-107 | — | E2E-015 |
| US-036.EC-1 | Intent ∧ stack exclusion | UT-107 | — | — |
| US-036.EC-2 | Zero eligible destinations honest empty | UT-108 | — | E2E-015 |
| US-037 | Availability honesty (reason hints, hide-only-irrelevant) | UT-097, UT-098 | — | E2E-018 |
| US-037.EC-1 | Unknown reason → generic honest fallback | UT-098 | — | — |
| US-037.EC-2 | Availability flap debounced | UT-103 | — | — |
| US-028.AC-5 | `program: true` in Go extension fails validate | UT-167 | IT-030 | — |
| US-038 | Programmable view authoring (React DX) | UT-169–UT-172 | IT-026 | E2E-031 |
| US-038.EC-1 | Starvation warning on blocking handler | UT-172 | — | — |
| US-038.EC-2 | Program throw → unavailable frame, fresh reopen | UT-165 | IT-027 | E2E-032 |
| US-038.EC-3 | `complete:true` local filter + stale-echo discard | UT-163 | — | — |
| US-038.EC-4 | Extension restart destroys view sessions | UT-162 | IT-027 | — |
| US-039 | Live views instant/honest under slow programs | UT-168 | IT-028 | E2E-031, E2E-032 |
| US-039.AC-3 | Circuit-break after 3 misses; shell stays live | UT-165 | IT-028 | E2E-032 |
| US-039.AC-4 | Per-client session isolation | UT-162 | IT-029 | — |
| US-039.EC-1 | Crash isolation (palette + other views live) | — | IT-027 | E2E-032 |
| US-039.EC-2 | Connection loss → degraded contract | UT-165 | — | — |
| US-039.EC-3 | Teardown on palette close; no orphan sessions | UT-162 | IT-026 | — |
| `cmdpalette` registry/catalog | Assembly, revision, uniqueness, degraded sources | UT-001–UT-005 | IT-001 | — |
| `cmdpalette` invoke | Validation, routing, statuses, single-flight | UT-006–UT-010 | IT-009, IT-010 | — |
| `cmdpalette` ranking | Match kinds, decay, transitivity, sections | UT-020–UT-030 | — | — |
| View vocabulary + patch | Validate, sanitize, requires/fallback, fencing | UT-040–UT-046 | IT-019 | — |
| Extension manifest/projection | Validation errors, health gates, overlay | UT-055–UT-063 | IT-016–IT-018 | — |
| Keymap/aliases evolution | Open ids, ext tier, conflicts, tolerant read | UT-070–UT-077 | IT-013, IT-014 | — |
| Config/settings sections | Defaults, validation, diff/apply, lifecycle | UT-080–UT-083 | IT-013–IT-015 | — |
| Personalization store | Upserts, pruning, isolation, corruption, reset | UT-088–UT-094 | IT-006, IT-011, IT-012 | — |
| Web projection/context | Hydration, SWR, predicates, exemptions | UT-095–UT-104 | — | — |
| Dispatch seam | client_op vs daemon routing, stale targets | UT-105–UT-108 | — | — |
| Search blend (web) | Group precedence, waves, caps, labels | UT-110–UT-117 | — | — |
| Args/confirmation | State machines, guards | UT-118–UT-124 | — | — |
| Action panel | Registration, filter, guards | UT-125–UT-129 | — | — |
| View renderers | Four kinds + unavailable frame | UT-130–UT-139 | — | — |
| Fallback | Row assembly, targets, spawn wiring | UT-140–UT-144 | IT-015 | — |
| Menubar projection | Item derivation, curation | UT-145–UT-147 | — | — |
| Settings UI | Table, alias cell, global section, recorder copy | UT-148–UT-151 | — | — |
| `packages/ui` Command custom filter | `shouldFilter={false}` + external order | UT-152–UT-154 | — | — |
| Desktop policies | Accelerator conversion, restore-on-fail, detection | UT-155–UT-158 | — | E2E-027–E2E-030 |
| Feedback lifecycle | Async progress/toast/retry | UT-159–UT-160 | IT-010 | E2E-014 |
| View-program runtime (daemon+web) | Gate, sessions, counters, quarantine, circuit-break | UT-161–UT-168 | IT-026–IT-030 | E2E-031–E2E-033 |
| React kit (`@compozy/extension-react`) | Reconciler→frames, id stability, patches, starvation | UT-169–UT-172 (vitest in `sdk/react`) | IT-026 | — |
| View-session routes (`open`/`events`/`DELETE`) | Per `_dx.md`/Part II shapes | UT-162 | IT-026, IT-029 | E2E-031 |
| Attached-client contract | Context snapshots, targeting, multiple_clients | UT-100, UT-173 | IT-031 | — |
| Workspace resolution boundary (L-033) | ID/name/path resolver + trusted native binding + isolation | — | IT-032 | — |
| Event admission (ADR-008 caps) | Seq supersession, cancel, caps, late disposal | UT-173 | IT-028 | — |
| Approval-pending single-flight lifetime | Guard held to terminal outcome | UT-009, UT-010 | IT-010 | E2E-024 |
| `GET /api/cmd-palette/commands` | Success + workspace scoping | — | IT-001, IT-002 | E2E-022 |
| `POST …/invoke` | 200/202/404/409/412×2/422 per `_dx.md` | UT-006–UT-010 | IT-009, IT-010 | E2E-023, E2E-024 |
| `GET/DELETE …/personalization` + `POST …/usage` | Snapshot/reset/recording | UT-092–UT-094 | IT-006, IT-008, IT-012 | E2E-025 |
| `GET …/stream` + views + patches | Fence, epoch guard, resync | UT-046 | IT-024, IT-019 | — |
| Settings sections (`cmd-palette`, `window-manager` ext) | GET/PATCH, 409/422 shapes | UT-080–UT-083 | IT-013–IT-015 | E2E-016 |
| CLI `compozy cmd-palette *` | Transcripts + exit codes per `_dx.md` | — | — | E2E-022, E2E-023, E2E-025 |
| Native tools `compozy__cmd_palette_*` | Args/result parity + catalog contract | UT-011 | IT-021 | E2E-026 |
| Manifest build/validate errors | Exact field-naming messages | UT-055–UT-062 | IT-016 | E2E-021 |
| Migration `00069` + fragment | Fresh/reopen/ahead/integrity/equivalence | — | IT-020 | — |
| Transport parity | New routes on HTTP+UDS | — | IT-022 | — |
| SSE catalog invalidation | Event → refetch → revision converge | — | IT-023 | E2E-020 |

## Unit Tests

### Registry & catalog (`internal/cmdpalette`) (Spec: System Architecture, Core Interfaces)

- **UT-001** (happy): `Registry.Catalog` — given corecmds + one extension provider, returns every descriptor resolved with bindings/alias/availability and a stable sha256 `Revision`.
- **UT-002** (happy): every core window-manager action id present in the catalog exactly once; count matches the corecmds inventory (absorption checklist assertion).
- **UT-003** (state): `Catalog` recomputes `Revision` when any descriptor field, source status, binding, or alias changes — and does NOT change it when only client-resolved availability changes (structural revision; availability rides `context_revision`); identical inputs → identical revision.
- **UT-004** (error): duplicate core id at boot → constructor returns `ErrDuplicateCommandID` naming both sources; duplicate `ext.*` id at load → later registration rejected with diagnostic containing the extension name.
- **UT-005** (error): a provider returning an error marks its source degraded in the catalog metadata; other sources still serve (never a silently shorter list).
- **UT-006** (error): `Invoke` on a `When`-failing command returns `ErrUnavailable{Reason}` with the byte-identical reason string the catalog carries.
- **UT-007** (error): `Invoke` with args violating the declared schema returns `ErrInvalidArguments{Fields: {"title": "required"}}` without executing.
- **UT-008** (error): `Invoke` unknown id returns `ErrCommandNotFound`; no fuzzy resolution occurs.
- **UT-009** (concurrency): two concurrent `Invoke` calls on a command declared `Policy.SingleFlight` — second returns `ErrAlreadyRunning`; a command declared parallel/retry-safe runs both; each declared execution-policy class is covered (no implicit "toggle" convention).
- **UT-010** (state): `Invoke` on a destructive tool-action command produces `approval_pending` through the tools policy fake; result carries `ApprovalID`; no execution before approval.
- **UT-011** (happy): native-tool handler `compozy__cmd_palette_list` returns the same command set as `Catalog` for the same workspace (parity assertion).
- **UT-012** (happy): settings-destination descriptors resolve `navigate` targets for every settings page in the route inventory.

### Ranking scorer (single TS implementation, `web/src/systems/os/lib/ranking/` — headless vitest; golden fixtures shared with the daemon's weight table) (Spec: Core Interfaces; ADR-003 as amended)

- **UT-020** (happy): `Rank` — "nwt" matches "New tab" via word-boundary/subsequence kind with its documented base-score band.
- **UT-021** (happy): diacritic + case folding — "sessao" matches "Sessão"-labeled candidate.
- **UT-022** (error): a two-term query where one term matches nothing drops the candidate entirely.
- **UT-023** (happy): exact-alias match outranks every non-exact result; alias owner beats a same-string title match.
- **UT-024** (happy): context boost — command matching the focused-session predicate outranks an equal-text global command.
- **UT-025** (ordering): comparator property test — for randomized candidate sets, ordering is a total order (transitive, antisymmetric) and stable across shuffles.
- **UT-026** (boundary): query of regex metacharacters `(*\` ranks as literal substring/subsequence; no panic, no empty-by-accident.
- **UT-027** (happy): frecency — same-text candidates, one with 10 uses today, ranks first; `DecayFrecency` at exactly one half-life returns weight/2.
- **UT-028** (boundary): decay at 120 days below prune threshold → candidate contributes zero and is flagged prunable.
- **UT-029** (happy): query learning — recorded ("gh" → id X) then `Rank("gh")` boosts X above tier boosts; newer conflicting association for Y dominates deterministically.
- **UT-030** (happy): `AssembleSections` empty-query — pins (pin-time order) → recents (last_used desc, palette-open excluded) → curated; with query: fixed group precedence, per-subtype promotion floors respected.

### View vocabulary & patches (`internal/cmdpalette/view*`) (Spec: Core Interfaces; ADR-007)

- **UT-040** (happy): a valid `list` view payload round-trips validate → normalized model with sections/rows/chips/empty intact.
- **UT-041** (error): detail rich text over the size cap is truncation-marked at Go wire validation; oversized/malformed payload fields reject with path-naming errors. (Rendered hostile-content behavior — sanitization to inert text, plain-text degradation — is owned solely by UT-136 in the web renderer suite.)
- **UT-042** (happy): row `Requires{"adaptiveGrid":"2"}` unmet with `Fallback: drop` removes the row; with a fallback element, renders the fallback; telemetry counter increments.
- **UT-043** (error): payload failing schema (unknown field type) returns a validation error naming the path `sections[0].rows[2].badge.tone`; served views never bypass validation.
- **UT-044** (boundary): payload with rows over the mount cap truncates with exact `showing N of M`; detail body over size cap truncation-marked.
- **UT-045** (error): unknown `kind: "canvas"` — authored manifest fails validate; runtime envelope renders the honest unknown-kind frame.
- **UT-046** (ordering): `ViewPatch` with `From` ≠ current revision is rejected and triggers resync request; in-order patch applies and advances revision.

### Extension manifest & projection (`internal/extension`) (Spec: Extensibility Integration Plan)

- **UT-055** (happy): manifest with the `_dx.md` `cmd_palette` block validates; ids namespace to `ext.notes.capture`; keywords/section/icon preserved; an authored `execution` policy block round-trips to the catalog/inspect projection, and an omitted one resolves the documented action-kind defaults.
- **UT-056** (error): command id `capture` declared twice → validate error `cmd_palette.commands[1].id: duplicate "capture"`.
- **UT-057** (state): `Manager.CmdPalette(ws)` includes contributions for enabled instances; disable/remove empties them; an unhealthy instance keeps its last-known validated descriptors with source status `unhealthy` + reason (membership ≠ health).
- **UT-058** (error): action referencing tool `purge_archved` (unknown) → validate error naming the field; unhealthy backing tool at runtime → command resolved `available:false` with availability reason.
- **UT-059** (error): title of 10 000 chars / control chars → validate rejects with length/charset rule.
- **UT-060** (state): crash-looping instance keeps its descriptors with `available:false`, source status `unhealthy`, and the crash-loop reason; recovery restores availability under a new revision; user overrides on its bindings survive dormant throughout.
- **UT-061** (state): dev-linked instance shadows published in its workspace; other workspace unaffected.
- **UT-062** (error): `default_shortcut: "meta+Küche"` → validate error naming the chord grammar.
- **UT-063** (state): dev reload with a broken manifest keeps last-good projection and records the error in dev diagnostics.

### Keymap & aliases (`internal/windowmanager`, `internal/cmdpalette`) (Spec: Config Lifecycle; ADR-005/006)

- **UT-070** (happy): `EffectiveKeymap` accepts an override for `ext.notes.capture` when the id exists in the registry `BindableIDs`; unknown id rejected as today.
- **UT-071** (error): binding `meta+KeyN` (owned by `session.new`) returns a conflict naming `session.new`; explicit-overwrite path unbinds the loser and flags it.
- **UT-072** (happy): shadow classification extends to new surface-local keys; `meta+KeyB` still reports `shadowed` with composer Bold.
- **UT-073** (error): bare `KeyD` binding rejected for non-exempt commands with the typing-guard reason.
- **UT-074** (state): stored override for a deleted command id is dropped on read with a diagnostic; boot never fails.
- **UT-075** (happy): ext default `alt+shift+KeyN` binds when free; appears in effective keymap attributed to the extension tier.
- **UT-076** (state): ext default colliding with a core default stays dormant with `conflict_with: <owner>`; two extensions same free chord — first enable wins, second dormant.
- **UT-077** (state): disabling the extension deactivates its bindings; a user override for its command persists and reactivates on re-enable.

### Config & settings sections (`internal/config`, `internal/settings`) (Spec: Config Lifecycle)

- **UT-080** (happy): `CmdPaletteConfig` defaults — `fallback_targets=["agent"]`, `personalization=true`; `Validate()` passes.
- **UT-081** (error): alias `"my alias"` (whitespace) and 33-char alias → `ValidationError{Path:"cmd_palette.aliases[...]"}` with the 1–32/no-whitespace rule.
- **UT-082** (error): `fallback_targets=["telegram"]` (unknown target) → validation error listing allowed targets.
- **UT-083** (happy): `SectionCmdPalette` build/diff/apply — PATCH toggling `personalization` writes only that path via the overlay editor; lifecycle class is `Live`.

### Personalization store (`internal/cmdpalette` + globaldb) (Spec: Data Models)

- **UT-088** (happy): `RecordUsage` upserts `(workspace, command)` incrementing `use_count`, accumulating decayed weight, updating `last_used_at`.
- **UT-089** (state): snapshot read prunes rows whose command id is absent from the catalog (usage, query hits, pins, aliases) tolerantly.
- **UT-090** (error): unreadable/corrupt personalization rows → snapshot degrades to zero-signal defaults, logs once, never errors the catalog call.
- **UT-091** (state): snapshot for workspace A contains no rows from workspace B (query-level isolation assertion).
- **UT-092** (state): the usage recorder's redaction path — given an invocation carrying argument values (incl. password-typed), the produced `Usage` record contains only the normalized pre-selection query (behavioral unit on the recorder; **IT-008 is the sole owner of the storage-level invariant**).
- **UT-093** (idempotency): double `Pin` for the same command keeps one row with the original `pinned_at`.
- **UT-094** (happy): `ResetPersonalization` clears all three tables for the workspace only; other workspaces untouched.

### Web projection & context (`web/src/systems/os`) (Spec: System Architecture)

- **UT-095** (happy): hydration hook renders last-known catalog immediately (SWR) and reconciles to the fetched revision without flicker.
- **UT-096** (state): daemon-unreachable — action commands render disabled with "runtime unavailable"; `AvailabilityExempt` commands stay enabled.
- **UT-097** (state): fully-irrelevant commands (feature off, wrong surface) absent from results; partially relevant render disabled-with-reason (US-037 split).
- **UT-098** (state): disabled rows expose the daemon reason verbatim in the hint slot; unknown reason renders the generic fallback copy.
- **UT-099** (happy): globe toggle widens every entity domain in one state change and persists via the shared preference write (single-flight).
- **UT-100** (state): context evaluator — focused-window predicates flip availability/boost on focus change; stale window id degrades to disabled, never targets a dead window.
- **UT-101** (state): evaluator with no daemon context never defaults to allow-all; context-dependent commands disable with reason.
- **UT-102** (happy): chord badges re-render from the effective keymap after a PATCH round-trip; no stale chord persists.
- **UT-103** (boundary): availability flapping 5× in one burst renders at most one enabled↔disabled transition (debounce).
- **UT-104** (happy): the palette stream consumer subscribes via a NAMED event listener (`addEventListener("cmd_palette.catalog.changed", …)` through `createStreamEventSource` — never bare `onmessage`, per L-017) and one event triggers exactly one refetch with revision convergence.

### Dispatch seam (`web/src/systems/os`) (Spec: System Architecture)

- **UT-105** (happy): `client_op` action routes to the window coordinator; `tool` action POSTs invoke; `navigate` pushes the app route; `url` opens external via the sanctioned opener.
- **UT-106** (error): invoking against a stale entity target surfaces the honest "no longer exists" error and requests list refresh.
- **UT-107** (state): destination intent set → view stack resets, and vice versa (mutual exclusion preserved through the new dispatch).
- **UT-108** (state): destination mode with zero eligible targets renders the honest empty state; Esc clears the intent.

### Search blend, sections, ghost (web) (Spec: Key Decisions)

- **UT-110** (happy): typing 2 chars fires gated domain queries; results integrate per-section as each resolves.
- **UT-111** (boundary): 257-char query → zero results + fallback row; input not truncated.
- **UT-112** (state): vault provider maps names/metadata only; no value field exists in the projected row type (type-level + runtime assertion).
- **UT-113** (boundary): entity section at cap renders exactly `cap` rows + "showing N of M"; grid/list views over mount cap virtualize/scroll with honest counts.
- **UT-114** (ordering): late async wave merges below already-rendered higher-precedence groups; keyboard selection survives via nearest-neighbor rule.
- **UT-115** (error): one domain adapter rejecting renders that section's inline error naming the domain; siblings unaffected.
- **UT-116** (happy): global-scope rows carry workspace labels; current-scope rows don't.
- **UT-117** (happy): "shortcuts" query yields the cheatsheet command in Commands and the settings page in Settings — both groups, fixed precedence.
- **UT-118** (happy): ghost tail renders when top-result confidence clears threshold; → at end-of-input accepts it into the query.
- **UT-119** (boundary): ghost preserves operator casing of the typed prefix; → mid-input only moves the caret; ambiguous results render no ghost.

### Inline args & confirmation (web) (Spec: `_uiux.md` S8/S9)

- **UT-120** (happy): selecting the `_dx.md` capture command morphs the bar into `title`/`tag` fields; ⇥ traverses; ⏎ with required filled executes.
- **UT-121** (error): ⏎ with `title` empty blocks and focuses it; pasted type-invalid value shows the field message; Esc restores search with values discarded.
- **UT-122** (state): bound-hotkey activation opens the palette directly in argument mode for that command.
- **UT-123** (happy): destructive command renders its declared confirmation with Cancel focused; Esc returns without executing.
- **UT-124** (state): held ⏎ from the trigger cannot confirm (repeat guard); target invalidated between trigger and confirm shows the invalidation message instead of executing.

### Action panel (web) (Spec: `_uiux.md` S7)

- **UT-125** (happy): ⌘K opens the panel for the selected row: sections, per-action chord badges, primary marked ↩; typing filters actions; ⌘K again closes.
- **UT-126** (happy): every command row's panel contains Pin/Unpin, Set alias…, Set shortcut…; entity rows list their domain actions with destructive styling.
- **UT-127** (state): row removed by live refresh while its panel is open → panel closes, selection falls to neighbor, no action fires.
- **UT-128** (state): panel on a disabled row lists meta-actions + the reason; unavailable domain actions are not rendered as runnable.
- **UT-129** (concurrency): action shortcut fires via the capture-phase listener when list focus drifted; `event.repeat` ignored.

### View renderers (web) (Spec: `_uiux.md` S2–S6)

- **UT-130** (happy): each kind (list/detail/form/grid) mounts under identical stack chrome; per-level search + ⌫-pops-on-empty + Esc-closes-all hold for all kinds; re-push mounts fresh (depth-keyed).
- **UT-131** (state): unknown/disabled view id renders the unavailable frame naming the extension; pop and reopen-at-root work; slow source shows loading then timeout with retry; Esc always exits.
- **UT-132** (boundary): (existing `palette-view-stack` suite extended) breadcrumb at depth 5 keeps ≤3 slots left-truncating.
- **UT-133** (happy): domain-view template renders chips with truthful counts, single-select, one-keystroke clear; the tasks exemplar's rendered state badges equal the shared status-tone dictionary's output for the same states (behavioral tone parity — topology itself is owned by the boundaries/lint gate, not a test).
- **UT-134** (state): empty-with-filter names the active filter; cold cache renders loading, never empty-flash.
- **UT-135** (state): detail pane follows selection without stealing focus; clears on row deletion; long content scrolls pane-only.
- **UT-136** (error): extension detail markdown with hostile content renders sanitized (integrates UT-041 fixture through the renderer).
- **UT-137** (happy/error): form renders declared fields in order; required-empty submit blocks with inline messages + first-invalid focus; empty dropdown shows declared hint; pop discards values.
- **UT-138** (error): submit failing downstream keeps the form open with the error and values preserved; password fields masked and cleared on pop.
- **UT-139** (happy/boundary): grid 2D arrow navigation across sections; ⏎/panel parity with list rows; media failure → placeholder tile; empty grid uses list empty grammar.

### NL fallback (web) (Spec: `_uiux.md` S1; F8)

- **UT-140** (happy/boundary): zero-match non-empty query appends "Ask agent: '{query}'" visually distinct row; empty query renders no fallback; weak-match boundary pinned against `WeightsV1.fallback_weak_match_threshold` (score at threshold → results + fallback together; below → fallback only).
- **UT-141** (state): no network call carries the query until ⏎ (spy on transport); weak-match threshold shows results + fallback together.
- **UT-142** (state): unresolvable default agent → ⏎ opens agent selection, then proceeds to session creation.
- **UT-143** (error): spawn failure surfaces the reason toast and preserves the query for retry.
- **UT-144** (idempotency): double-⏎ within the repeat guard creates one session; deliberate repeat later creates a distinct session.

### Menubar projection (web) (Spec: `_uiux.md` S11)

- **UT-145** (happy): a curated menu definition of registry ids renders labels + chords + availability from the projection; dispatch goes through the same seam as the palette.
- **UT-146** (state): item whose command becomes unavailable renders disabled with the same reason; item never vanishes mid-session.
- **UT-147** (state): disabling an extension removes its curated-in items at next render without breaking the menu.

### Settings UI (web) (Spec: `_uiux.md` S12/S15/S16)

- **UT-148** (happy): shortcut table lists the full registry with source filter (Core areas / per-extension); recorder assigns; conflict blocks naming the culprit with explicit overwrite flow flagging the loser.
- **UT-149** (happy/error): alias cell edits inline; invalid alias shows the 1–32/no-whitespace rule; saved alias appears on palette rows as "Title (alias)".
- **UT-150** (state): global-hotkey section — browser mode renders disabled with the requires-desktop-shell reason; failed registration renders the in-use-by-another-application state with the previous binding still effective. (The non-QWERTY limitation is owned by the P8 QA-scenario walk as a user journey, not a prose assertion.)
- **UT-151** (happy): cheatsheet groups extension bindings by source and shows every effective chord incl. multi-chord display rules; Palette settings section toggles the agent fallback (off → no fallback row renders) and resets personalization with scope confirmation.

### `packages/ui` Command (custom-filter path) (Spec: Known Risks)

- **UT-152** (happy): `Command` with `shouldFilter={false}` renders externally-ordered items verbatim (no cmdk re-sort).
- **UT-153** (happy): keyboard selection (`ArrowDown`+`Enter` → `onSelect`) works with external filtering and live item replacement.
- **UT-154** (state): selection survival helper keeps the highlighted value when the item set churns and falls to nearest neighbor when removed (story + play-function twin).

### Desktop policies (`desktop/src`) (Spec: Integration Points)

- **UT-155** (happy): chord→accelerator conversion table — `meta+shift+Space` → `CommandOrControl+Shift+Space`, `alt+shift+KeyN` → `Alt+Shift+N`; unconvertible chords return a typed error.
- **UT-156** (state): `globalShortcuts.sync` policy — failed registration returns `failed_in_use` and restores the previously registered binding; quit unregisters all.
- **UT-157** (state): macOS accessibility requirement detected at bridge init produces the `failed_permission` status with the settings deep-link payload.
- **UT-158** (happy): bridge contract validator — method allowlist + params validation mirrors `boot-contract.ts` discipline (unknown method rejected).

### Feedback lifecycle (web) (Spec: `_uiux.md` S14)

- **UT-159** (happy): sync client_op closes the palette with visible effect; async invoke shows in-palette pending, then success toast via `notifyUser` after close; cross-workspace landing toast names the workspace.
- **UT-160** (error): invoke failing mid-flight (transport rejection) produces the failure toast naming command + reason with retry only for idempotent-safe commands.

### View-program runtime (daemon + web) (Spec: Core Interfaces `view_program.go`; ADR-008)

- **UT-161** (error): `view/open` for an extension without granted/implemented `view.provider` fails through the standard service gate with the same error class as other surfaces; healthy TS fixture opens and returns a first frame.
- **UT-162** (state): session registry — `open` mints a session bound to (client, extension instance); events for another extension's session are rejected (ownership check); `close` is idempotent and cancels in-flight work; extension restart (generation bump) invalidates all its sessions; two clients opening the same view get independent sessions; palette dismiss tears the session down (no orphans).
- **UT-163** (ordering): event-counter discipline — host increments per local edit; a frame echoing a stale counter is applied to the model but never to the focused input; `complete:true` result sets switch the host to local filtering until the next reset.
- **UT-164** (state): handler quarantine — ids absent from the latest frame's `Handlers` survive two frames; events on quarantined ids drop silently; events on unknown ids log a warning, never error.
- **UT-165** (state): degradation state machine — soft-budget expiry shows busy with previous rows; hard-ack miss → `degraded` with last-good + retry; third consecutive miss → circuit-broken until reopen; program throw/connection loss enter the same machine with their reasons; Esc/⌫ remain handled throughout (assert shell handlers fire while degraded).
- **UT-166** (ordering): `view/patch` push fan-out — patch with matching `From` revision applies and advances; mismatch triggers full-payload resync request; out-of-order patches never apply (extends UT-046 to the push path).
- **UT-167** (error): manifest validation — `program: true` in a Go-toolchain extension fails validate with the exact message from `_dx.md`; `program: true` without `view.provider` implemented fails at build describe-coverage.
- **UT-168** (boundary): band budgets (injected clock) — busy affordance appears only after the soft budget; previous rows retained; hard-ack timer resets on any frame including `view.status`.
- **UT-173** (concurrency): event admission — event seq N+1 arrives while N is in flight → N's request receives the cancel (SDK abort observed) and its late frame (`InReplyTo=N`) is discarded even when it carries a newer revision; a canceled handler that ignores its abort, mutates state, and emits an `InReplyTo=0` push carrying its superseded generation → **rejected** (causal-generation rule); independent action events beyond the per-session cap are rejected with the structured busy error; frames after `view/close` or an extension generation bump are disposed by session-id validity.

### React kit `@compozy/extension-react` (vitest in `sdk/react`) (Spec: Extensibility Integration Plan; ADR-009)

- **UT-169** (happy): rendering the `_dx.md` `NotesBrowser` example against a mock transport produces a valid `list` frame (sections/rows/chips/actions) through the persistent-mode reconciler.
- **UT-170** (state): re-render with changed row set keeps handler ids stable for surviving actions (same id → new closure), emits removed handlers for quarantine, and never churns ids on no-op renders.
- **UT-171** (ordering): a burst of `setState` calls within one frame-clock tick emits exactly one patch; content-hash dedupe suppresses identical frames; patches apply cleanly over the previous frame (round-trip property test).
- **UT-172** (error): a handler blocking the event loop past the yield budget triggers the starvation warning in dev diagnostics; `AbortSignal` delivered to handlers aborts in-flight `useCachedPromise` work on session close.

## Integration Tests

### `/api/cmd-palette` family (real globaldb, both transports where noted)

- **IT-001**: `GET /api/cmd-palette/commands?workspace=acme` — seeded corecmds + fixture extension → 200 with every command, bindings, aliases, `catalog_revision`; `-o jsonl` CLI path streams complete at 500+ commands.
- **IT-002** (withdrawn): entity/settings search is web-owned (Part II Key Decisions — client-side over existing domain queries); the invariants live in UT-110/UT-112 (web) + E2E-003; Go integration owns only the real domain endpoints' auth/scope contracts in their own canonical suites.
- **IT-003**: scope preference round-trip — PATCH the shared scope pref, `GET` from a second client sees it (daemon persistence).
- **IT-004** (withdrawn): built-in domain-view assembly is web-owned (TS view model over TanStack queries); the invariants live in UT-133/UT-134 (web) + E2E-009 (150-cap + truthful counts walked in the browser).
- **IT-005**: form-declared command submit → tool invocation with mapped args → result envelope.
- **IT-006**: usage → snapshot loop — `POST /usage` ×10 for `session.new` across 3 days (injected clock) → `GET /personalization` shows expected weights; recents order matches `last_used_at`; pins included.
- **IT-007**: invoke with inline args — `POST …/invoke {args:{title:"Standup follow-ups"}}` → 200 with tool result (fixture tool echo).
- **IT-008**: usage recording for a password-arg command — tables contain the query only (SQL-level assertion).
- **IT-009**: invoke policy matrix — unknown id → 404 `command_not_found`; schema-invalid → 422 with fields; unavailable → 412 with reason; destructive → 202 `approval_pending` + approval flow completion; denial → structured failure, no side effect.
- **IT-010**: single-flight + failure — concurrent invokes → one 200 one 409; daemon-side tool crash → failure result with reason; retry succeeds. Approval lifetime: duplicate invoke while `approval_pending` → 409; deny and timeout each release the guard (next invoke proceeds); approval completing after the original client disconnected resolves exactly once with no duplicate effect.
- **IT-011**: corruption/isolation — corrupt a personalization row → snapshot degrades with defaults + one log; workspace B snapshot unaffected by A's data.
- **IT-012**: `DELETE /personalization?workspace=acme` → 200 reset; subsequent GET empty; workspace B intact.

### Settings sections

- **IT-013**: `PATCH /api/settings/window-manager` binding `ext.notes.capture → meta+shift+KeyN` → 200 effective map; conflicting chord → 409 `shortcut_conflict` naming owner; unknown-id override in config tolerated on GET with diagnostic; concurrent PATCHes converge last-write-wins.
- **IT-014**: aliases via the same PATCH — set/clear round-trip; invalid grammar → 422 `invalid_alias`; alias visible in `GET /api/cmd-palette/commands`.
- **IT-015**: `GET|PATCH /api/settings/cmd-palette` — agent-fallback toggle + personalization toggle apply Live (no restart record); values echo in config surface (`compozy config get cmd_palette.personalization`); unknown fallback target value → 422 naming the allowed set.

### Extension contribution (fixture extension)

- **IT-016**: enable fixture → commands/views/default-chord appear in catalog with new revision; disable → gone atomically (one revision step); core-id collision fixture rejected at load with diagnostic; dormant default listed with conflict owner.
- **IT-017**: health vs membership — kill fixture subprocess (crash loop) → contributions REMAIN in the catalog with source status `unhealthy` + reason and `available:false`; disable → membership gone; recovery → available again under a new revision; untrusted-workspace fixture → commands `available:false` with trust reason; degraded extension subsystem reported per-source in `sources`.
- **IT-018**: dev overlay + reload — dev-link fixture shadows published in its workspace; edit manifest → projection updates within watch interval; broken edit keeps last-good + dev diagnostic.
- **IT-019**: extension view path — `GET /api/cmd-palette/views/ext.notes.recent` returns validated envelope; invalid payload from the fixture tool → error frame contract + extension diagnostic; patch stream with wrong `From` fence → resync full payload.

### Store, events, parity

- **IT-020**: migrations `00069` (tool_approval_pending, P1) and `00070` (cmd_palette_* tables, P2) — extend the canonical fresh/reopen/ahead/integrity/equivalence suites (`migrate_streams_test.go`) incl. workspace delete-trigger cascades; approval lifecycle races: crash between dispatch and completion recovers to `uncertain` (never re-executes); duplicate `Resolve` rejected by the fence.
- **IT-021**: native-tool catalog — `compozy__cmd_palette_list/invoke` descriptors present in `native-tool-catalog.json` contract check; invoke tool honors approval gate.
- **IT-022**: transport parity — every new `/api/cmd-palette/*` + settings route registered on HTTP and UDS (extend `transport_parity_integration_test.go`).
- **IT-023**: SSE invalidation — enable extension → `cmd_palette.catalog.changed` observed on `GET /api/cmd-palette/stream` with the new revision; event name registered in the events registry.
- **IT-024**: stream replay guards — view patch stream `after>0` without `stream_epoch` → 400; epoch mismatch → full resync envelope.
- **IT-025**: keymap live apply — PATCH shortcuts → windowmanager snapshot carries the new effective chord without daemon restart (Live lifecycle).

### View programs (TypeScript fixture extension)

- **IT-026**: full loop — `POST /views/ext.notes-ts.browser/open` binds the caller's client identity, mints a session + first frame + stream token; `POST /view-sessions/{s}/events` with a search handler id + event counter returns 202 and a patch arrives on the **session-scoped** stream; effects carry stable ids and, once acked via `ack_effects`, a forced resync re-delivers the frame WITHOUT re-executing them (at-most-once, SI-21); `DELETE` closes; reopening after palette dismiss finds no orphan session; a call with another client's stream token → ownership rejection (server-side).
- **IT-027**: crash + restart — kill the fixture mid-session → session invalidated, stream carries the unavailable state; supervisor restarts the extension → old session ids rejected, fresh `open` succeeds with a new first frame.
- **IT-028**: slow program — fixture configured to stall past the hard ack → stream carries `degraded`; three consecutive stalls → circuit-broken marker; a fast view on the same fixture stays unaffected (per-view isolation).
- **IT-029**: two clients — two `open` calls for the same view yield distinct sessions; events on one never produce frames on the other's stream.
- **IT-030**: validate matrix — Go fixture with `program: true` fails `compozy extension validate` with the exact `_dx.md` message; TS fixture missing `view/close` fails build describe-coverage naming the method.

### Client context & workspace resolution

- **IT-031**: two attached clients (shell + browser fixtures) with divergent focused windows — `GET /commands?client=A` and `?client=B` return different availability/reasons for focused-window commands; invoke without `client` → 409 `multiple_clients` listing both ids; invoke with `client=B` executes against B's context (re-evaluated: a command available on A but not B → 412 with B's reason); a stale attachment (client disconnected) is pruned from `/clients` and targeting it → 412 `no_attached_shell`. Authorization paths: a forged/mismatched `X-Compozy-Client-Token` on a self-originated call → structured rejection; a control-plane caller (UDS CLI, native tool) targets `client=B` successfully with NO token (privileged path); `catalog_revision` is identical for `?client=A` and `?client=B` (structural only) while availability differs.
- **IT-033**: canonical event matrix — every low-frequency `cmd_palette.*` event from the Monitoring matrix is registered in the events registry and emitted at its owning domain boundary with its required correlation fields (`invocation_id`/`approval_id` on `command.invoked`; `view_session` on session events; `effect_id` on effect-failure logs; workspace on all); keystroke/patch paths emit none (metrics only).
- **IT-032**: workspace resolution matrix — `--workspace` accepts canonical ID, name, and path (incl. nested cwd default with no flag) resolving to the same canonical ID across CLI, HTTP, UDS, and native-tool paths; two-workspace isolation: catalog, personalization, and view sessions of workspace A are unreachable through every workspace-B-scoped call; native tools bind the trusted session workspace and reject spoofed foreign refs.

## End-to-End Tests

### Operator — root, search, execution (US-001..008, 015..017, 037) — Playwright web

- **E2E-001**: ⌘K → type "nwt" → ghost completes "New tab" → ⏎ → new tab opens; row showed the ⌘N chord badge.
- **E2E-002**: open palette with 60+ commands + 200 sessions seeded → sections ordered, overflow note exact, keyboard nav responsive.
- **E2E-003**: type an agent name → Agents section loads async and lists it; kill the tasks endpoint fixture → Tasks section shows inline error; select a session deleted mid-open → honest "no longer exists".
- **E2E-004**: toggle globe → foreign-workspace session row with workspace label → ⏎ → workspace switches first, session window focused, toast names the switch.
- **E2E-005**: type "shortcuts" → both the cheatsheet command and Settings result render in their groups → settings result navigates to the shortcuts page.
- **E2E-006**: execute "New session" 3× → reopen: Recents lists it; pin it via panel → rest-state shows Pinned first; frecency places it above textual peers for "se".
- **E2E-007**: with a focused session window, open palette → session-contextual commands boosted; close the window (second client) → availability re-evaluates without selection jump.
- **E2E-008**: push Sessions view → push a session's detail → ⌫ on empty query pops one level with parent state intact → Esc closes all → reopen at root.
- **E2E-009**: open Tasks view → chips show truthful counts → select "Failed" with zero matches → honest empty naming the filter → one-keystroke clear.
- **E2E-010**: select an entity row → detail pane previews it → ⌘K → action panel with domain actions + meta-actions → run a secondary action via its badge shortcut.
- **E2E-011**: run the capture fixture command → inline args mode → ⏎ with empty required blocks and focuses field → fill → ⏎ → success toast; form-view variant preserves values on downstream failure.
- **E2E-012**: open the marketplace Grid view → arrow across tiles 2D → broken image shows placeholder → ⏎ opens the item.
- **E2E-013**: type gibberish → only "Ask agent" row → ⏎ → new session opens with the query as first prompt; with no default agent configured → picker appears first.
- **E2E-014**: invoke a slow async fixture command → in-palette pending → close palette → completion toast arrives (survives unmount); failure variant offers retry.
- **E2E-015**: new-tab destination flow → destination palette lists eligible targets only → pick → tab opens; zero-eligible variant shows honest empty and Esc exits clean.

### Operator — keyboard & settings (US-022..025, 034) — Playwright web

- **E2E-016**: Settings → Shortcuts: filter by the fixture extension → record `⌘⇧N` on its command → conflict with `session.new` blocks naming it → overwrite explicitly → loser flagged; set alias "cap" → palette shows "Capture note (cap)" ranked first for "cap"; cheatsheet reflects both immediately.
- **E2E-017**: disable the fixture extension while its view is open → unavailable frame names it → pop works → its commands and bindings gone; re-enable restores with user override intact.
- **E2E-018**: stop the daemon → palette stays open with disabled-with-reason rows, cheatsheet command still works (exempt) → restart → rows re-enable without reopening.
- **E2E-019**: menubar Window menu shows registry labels + chords; rebind close-window → menu chord updates without reload; unavailable item disabled with same reason as palette row.
- **E2E-020**: (SSE) enable the fixture extension from a second client (CLI) → first client's open palette gains its commands within the invalidation window.
- **E2E-021**: `compozy extension dev` on the fixture → edit a command title → open palette reflects it without reload; introduce a manifest error → last-good stays, `compozy extension logs` shows the validation error.

### Agent — control plane (US-032..034) — Go runtime harness, UDS, `_dx.md` transcripts verbatim

- **E2E-022**: `compozy cmd-palette list -o json` → full catalog with ids/availability/bindings/arguments; `--available=false -o jsonl` shows reasons; `--source ext.notes` filters.
- **E2E-023**: `compozy cmd-palette invoke ext.notes.capture --arg title="Standup follow-ups"` → `status:ok` + result; missing required → exit 2 invalid-arguments; unknown id → exit 1 not-found; `window.close` with no shell → `no_attached_shell` error text.
- **E2E-024**: `compozy cmd-palette invoke ext.notes.purge` → `approval_pending` + approval id; approve via the approvals surface → completion; deny → structured denial, no effect.
- **E2E-025**: `compozy cmd-palette personalization show/reset` transcripts; `compozy config set cmd_palette.personalization false` stops recording (verified via subsequent show); binding/alias parity — `bind` (conflict → named owner → `--overwrite` flags the loser), `alias set/clear`, `unbind`, and `bindings -o json` (effective + dormant + conflicts) exactly per the `_dx.md` transcripts.
- **E2E-026**: session agent calls `compozy__cmd_palette_list` then `compozy__cmd_palette_invoke` → parity with CLI results; destructive invoke surfaces the approval to the operator.

### Desktop shell (US-024) — Playwright `_electron`

- **E2E-027**: with the shell running and another window focused, fire `⌘⇧Space` → Compozy window focuses/restores with the palette open; variant: over an open modal, window focuses without executing through it.
- **E2E-028**: assign a global hotkey to an argument command → fire it unfocused → window summons with the palette in argument mode.
- **E2E-029**: simulate registration failure (pre-register the accelerator in the harness) → Settings shows "unavailable — in use by another application" and the previous binding still works; relaunch re-registers and re-reports.
- **E2E-030**: run the web app in plain browser mode → global-hotkey settings disabled with "requires desktop shell"; in-app ⌘K unaffected.

### Programmable extension views (US-038, US-039) — Playwright web + TS fixture

- **E2E-031**: open the fixture's program view → typing echoes instantly while live results refine; toggle a chip → counts update; push a row's detail → ⌫ returns with parent query/selection intact; submit the edit form → toast; destructive action → confirmation with Cancel focused → confirm → row gone.
- **E2E-032**: slow-mode fixture → previous rows + busy at soft budget; degraded with retry at hard ack; after three misses the view circuit-breaks while Esc, root palette, and a built-in view keep working; crash-mode variant shows the unavailable frame naming the extension.
- **E2E-033**: `compozy extension dev` on the TS fixture → edit the program while its view is open → "view reloaded" note, session reopens on next entry running the new code.

### QA scenarios (docs/qa)

- This spec resets/extends `ET-web-command-palette-shortcuts` and `ET-palette-nested-views`, re-walks `ET-palette-sessions-view-switch` (currently `blocked-verify`) in P8, and adds content-addressed `untested` scenarios for: registry-driven root, action panel, inline args + confirmation, extension contributions, agent invoke path, global summon. Walks follow the `qa-execution` contract before close.
