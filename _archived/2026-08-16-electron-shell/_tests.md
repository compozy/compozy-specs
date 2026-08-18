# Test Specification: Electron Desktop Shell Migration

Canonical test contract for the shell migration. Companion to `_spec.md`. Derived from `_user_stories.md` (behavior), `_spec.md` Part II (components), `_dx.md` (CLI/API journeys — transcripts are used verbatim), and `_uiux.md` (browser journeys).

## Strategy

- **Go**: standard `testing` with `t.Run("Should …")` subtests + `t.Parallel`, `-race`, table-driven; fakes only at I/O boundaries (release feed = local mock GitHub server fixture; filesystem = `t.TempDir` homes). Canonical suites extended in place: `internal/update/*_test.go`, `internal/cli/update_command_test.go`, `internal/cli/app_test.go`, `internal/desktoprelease/release_test.go`, settings handler suites under `internal/api`.
- **Web**: Vitest + Testing Library, run via `bunx turbo run test --filter=./web`; canonical suites extended (`-general.test.tsx` family for S1, os-menubar suite for S2). No snapshot/CSS-literal tests.
- **Shell (TypeScript main process)**: Vitest colocated policy tests in `desktop/` (reference pattern: pure, Electron-free modules per `.resources/hermes` discipline) — window-state clamp, deep-link validation, operation-record consumption, nav guards. Boot-ladder supervision policy (resolve/probe/retry/give-up) is **Go-owned** (B-001) and unit-tested in Go; the TS side tests only bootstrap download/verify and rendering adapters.
- **Integration**: `make test-integration` (+`integration` tag) — daemon apply/restart/rollback against the mock feed; HTTP/UDS parity.
- **E2E**: new `make test-e2e-desktop` lane — Playwright `_electron` driving the real packaged shell against a real daemon and mock feed; browser lane extends `make test-e2e-web` for `_uiux.md` surfaces; CLI journeys assert the exact `_dx.md` transcripts.
- **Pipeline gates (PIPE)**: behaviors enforced by CI job structure (inventory assertion, publish ordering, packaged smoke) are annotated `PIPE` in the matrix — they gate releases, not test suites. **QA gates (QA)**: the recorded macOS+Linux `APP-*` walks and the real beta N→N+1 cycle.

## Coverage Matrix

| Source | Behavior | Unit | Integration | E2E |
| --- | --- | --- | --- | --- |
| US-001 | Fresh install first run provisions | UT-053 | — | E2E-001 |
| US-001.EC-1 | Corrupt/missing bundle payload → refusal + reinstall guidance | UT-054 | — | E2E-002 |
| US-001.EC-2 | Bundled binary integrity failure refused | UT-055 | — | E2E-002 |
| US-001.EC-3 | Disk exhausted → named cause, retry works | UT-054 | — | — |
| US-001.EC-4 | Runtime newer than supported → refuse attach | UT-056 | — | E2E-003 |
| US-001.EC-5 | Double-launch during first run → one instance | — | — | E2E-004 |
| US-002 | In-place generational replacement | — | — | QA (`APP-install-first-run-provision` walk) |
| US-002.EC-1 | Old AppImage alongside → status reports new | UT-027 | — | QA |
| US-002.EC-2 | Old app running during install → no corruption | — | — | QA |
| US-002.EC-3 | Replacement mid-task → runtime untouched | — | — | QA |
| US-003 | Attach to running runtime | UT-057 | — | E2E-005 |
| US-003.EC-1 | Runtime below minimum → refuse + repair path | UT-056 | — | E2E-003 |
| US-003.EC-2 | Unhealthy listener → start/repair, no hang | UT-057 | — | E2E-006 |
| US-003.EC-3 | Stale runtime record → clean + start | UT-058 | — | E2E-006 |
| US-004 | Start installed stopped runtime | UT-057 | — | E2E-005 |
| US-004.EC-1 | Corrupt binary → diagnostics + phase | — | — | E2E-006 |
| US-004.EC-2 | Readiness never arrives → bounded retry + dialog | UT-059 | — | E2E-006 |
| US-004.EC-3 | Port conflict → named conflict | UT-057 | — | — |
| US-005 | Boot failure explains + retry/logs/quit | UT-059 | — | E2E-006 |
| US-005.EC-1 | Retry overlap → attempts never interleave | UT-060 | — | — |
| US-005.EC-2 | Repeated same-cause failures → bounded backoff | UT-059 | — | — |
| US-005.EC-3 | Healthy retry no-op, state uncorrupted | UT-033 | IT-017 | E2E-024 |
| US-006 | Quit leaves runtime running | — | — | E2E-007 |
| US-006.EC-1 | Quit during runtime apply → safe completion/rollback | — | IT-010 | E2E-007 |
| US-006.EC-2 | Quit during provision → clean resume next launch | UT-054 | — | — |
| US-006.EC-3 | OS shutdown → consistent state | — | — | QA (`APP-quit-contract`) |
| US-007 | Second launch focuses | — | — | E2E-004 |
| US-007.EC-1 | Rapid launches → one instance | — | — | E2E-004 |
| US-007.EC-2 | Unresponsive first instance → no duplicate | — | — | QA (`APP-single-instance-focus`) |
| US-008 | Window geometry restore | UT-061 | — | E2E-008 |
| US-008.EC-1 | Disconnected display → clamp/center | UT-061 | — | — |
| US-008.EC-2 | Corrupt state file → defaults, rewritten | UT-062 | — | — |
| US-008.EC-3 | DPI change → fully visible | UT-061 | — | — |
| US-009 | Zoom steps + persistence | UT-063 | — | E2E-009 |
| US-009.EC-1 | Zoom at bounds → no-op | UT-063 | — | — |
| US-009.EC-2 | Reload re-applies zoom | UT-063 | — | E2E-009 |
| US-010 | External links → system browser | UT-064 | — | E2E-010 |
| US-010.EC-1 | Non-web scheme → never executed | UT-064 | — | — |
| US-010.EC-2 | Hostile top-level navigation blocked | UT-064 | — | E2E-010 |
| US-011 | Browser/app render identically | — | — | E2E-011 |
| US-011.EC-1 | No GPU → software fallback, usable | — | — | QA (Linux VM charter) |
| US-011.EC-2 | Wayland vs X11 parity | — | — | QA (Linux charter) |
| US-012 | UI crash auto-recovery | UT-065 | — | E2E-012 |
| US-012.EC-1 | Crash loop → bounded, then dialog | UT-065 | — | — |
| US-012.EC-2 | Hang logged distinct from crash | UT-065 | — | — |
| US-013 | Deep link navigates running app | UT-066 | — | E2E-013 |
| US-013.EC-1 | Hostile payloads → default view | UT-066, UT-030 | — | E2E-014 |
| US-013.EC-2 | Unknown product path → SPA not-found | — | — | E2E-013 |
| US-013.EC-3 | Two links fast → last wins | UT-067 | — | — |
| US-014 | Deep link cold start | UT-067 | — | E2E-015 |
| US-014.EC-1 | Hostile cold-start payload → default view | UT-066 | — | E2E-015 |
| US-014.EC-2 | Not installed → deterministic CLI error | UT-031 | — | E2E-025 |
| US-014.EC-3 | Link during in-flight boot → single instance | UT-067 | — | — |
| US-015 | App update cycle | UT-006, UT-040 | IT-007 | E2E-016 |
| US-015.EC-1 | Feed unreachable → silent log + manual error | UT-007 | — | E2E-026 |
| US-015.EC-2 | Bad signature/integrity → refused, untouched | UT-004 | IT-003 | — |
| US-015.EC-3 | Equal/lower version → never offered | UT-003 | — | — |
| US-015.EC-4 | deb format → manual path explained | UT-008 | — | — |
| US-015.EC-5 | Two applies race → one proceeds | UT-011 | IT-010 | — |
| US-015.EC-6 | App+runtime mutual exclusion | UT-011 | IT-010, IT-018 | — |
| US-016 | Failed app update recovers | UT-068, UT-077 | IT-019 | E2E-017 |
| US-016.EC-1 | quitAndInstall never exits → watchdog marks failed | UT-068 | — | — |
| US-016.EC-2 | Crash mid-install → version truth next boot | UT-068 | — | E2E-017 |
| US-016.EC-3 | Repeated failures → needs-attention escalation | UT-068 | — | — |
| US-017 | Runtime update quiesce/swap/verify | UT-005 | IT-001 | E2E-018 |
| US-017.EC-1 | Compat gate: runtime requiring newer app → refused pre-mutation | UT-074 | IT-020 | E2E-018 |
| US-017.EC-2 | Interrupted swap → journal completes/rolls back | UT-009 | IT-002 | — |
| US-017.EC-3 | Externally managed runtime → recommendation | UT-008 | — | E2E-026 |
| US-017.EC-4 | Both tracks ordered, never concurrent | UT-011 | IT-010 | — |
| US-018 | Headless self-update, no app | UT-001 | IT-001 | E2E-027 |
| US-018.EC-1 | Concurrent with app-driven update → blocked | UT-011, UT-075 | IT-011, IT-018 | — |
| US-018.EC-2 | Unsupported platform → deterministic error | UT-010 | — | — |
| US-018.EC-3 | Headless rollback on failed health | UT-009 | IT-002 | — |
| US-018.EC-4 | Permission denied → no partial swap | UT-010 | — | — |
| US-019 | `[app]` config governs checks | UT-072 | IT-016 | — |
| US-019.EC-1 | Config change honored next check | UT-072 | — | — |
| US-019.EC-2 | Absent config → defaults | UT-072 | — | — |
| US-020 | `app status` schema-valid truth | UT-027 | — | E2E-024 |
| US-020.EC-1 | Corrupt/unknown schema record → deterministic error | UT-028 | — | — |
| US-020.EC-2 | Installed, no record → coherent report | UT-027 | — | — |
| US-020.EC-3 | Concurrent write → no torn read | UT-029 | — | — |
| US-021 | `app open` focus-or-launch | UT-030 | — | E2E-025 |
| US-021.EC-1 | Invalid path arg → `invalid_target_path` | UT-030 | — | E2E-025 |
| US-021.EC-2 | Dead socket → deterministic error within timeout | UT-032 | — | — |
| US-022 | Single `compozy update`, multi-target record | UT-019–UT-023 | IT-004 | E2E-026 |
| US-022.EC-1 | In-flight elsewhere → `blocked` + holder | UT-022 | IT-011 | — |
| US-022.EC-2 | App quits mid-apply → truthful interruption | UT-068 | — | — |
| US-022.EC-3 | No app → no `app` object | UT-020 | — | E2E-027 |
| US-022.EC-4 | Managed install → recommendation, exit 0 | UT-021 | — | E2E-026 |
| US-022.AC-5/EC-5 | Cancel: dormant succeeds, live lease declines, cancel/resume race single-winner | UT-083 | IT-024 | — |
| US-001.AC-6 | Offline first run (install-from-bundle) | UT-054, UT-055 | — | E2E-001 |
| US-023 | `app retry` re-drives boot | UT-033 | — | E2E-024 |
| US-023.EC-1 | Not running → `app_not_running` | UT-033 | — | — |
| US-023.EC-2 | Retry races in-flight boot → one attempt | UT-060 | — | — |
| US-024 | `app diagnose` + consent-gated bundle | UT-034 | — | E2E-024 |
| US-024.EC-1 | Unwritable bundle path → no partial archive | UT-034 | — | — |
| US-024.EC-2 | Diagnose mid-update → truthful in-flight state | UT-034 | — | — |
| US-024.EC-3 | Rotation boundary → consistent archive | UT-034 | — | — |
| US-025 | Publish pipeline gates | UT-069 | — | PIPE (build+smoke jobs) |
| US-025.EC-1 | Missing secret → preflight fails run | — | — | PIPE (preflight assertion) |
| US-025.EC-2 | Version ≤ live → refused | UT-070 | — | PIPE |
| US-025.EC-3 | Interrupt between payload/manifest → consistent channel | — | — | PIPE (ordered upload steps) |
| US-026 | Feed repair to known-good | UT-081 | — | E2E-033 |
| US-026.EC-1 | Repair vs publish race → one wins | — | — | PIPE (concurrency group `desktop-channel`) |
| US-026.EC-2 | GC'd rollback target → inventory error | UT-069, UT-081 | — | — |
| US-027 | Cutover completeness | UT-071 | — | QA (cutover checklist walk) |
| US-027.EC-1 | Abandoned install polls deleted feed | — | — | QA (observed error log, no server object) |
| US-027.EC-2 | New-generation feed repair only | — | — | PIPE |
| US-028 | Docs truth pass + Chromium-first | — | — | QA (docs review checklist) + link-check gate |
| US-028.EC-1 | Surviving old-shell reference = cutover defect | UT-071 | — | — |
| US-028.EC-2 | `[app]` docs unchanged; new keys documented | — | — | QA (docs review) |
| US-029 | Web update surface, both tracks | UT-040–UT-046 | IT-012 | E2E-019 |
| US-029.EC-1 | Managed → recommendation, no affordance | UT-042 | — | E2E-019 |
| US-029.EC-2 | Browser apply w/ app closed → staged copy | UT-044 | IT-008 | E2E-020 |
| US-029.EC-3 | No app installed → single-track section | UT-041 | — | E2E-019 |
| US-029.EC-4 | Applying/failed → indicator hidden | UT-050 | — | E2E-021 |
| US-029.EC-5 | Daemon restart mid-apply → truthful reconnect | UT-045 | IT-005 | E2E-020 |
| US-029.EC-6 | Refresh failure → last-known + retry | UT-046 | — | — |
| `internal/update` (Part II component) | Unified mechanism error paths | UT-001–UT-018, UT-074, UT-077 | IT-001–IT-011, IT-020 | — |
| Update Operation + coordinator (ADR-009) | Journal, recovery, exclusivity, lease, plan, events | UT-075, UT-076, UT-080, UT-083 | IT-018, IT-019, IT-023, IT-024 | E2E-031 |
| Settings surface (Part II component) | GET/POST both transports, full projection | UT-035–UT-039, UT-079, UT-082 | IT-012–IT-014 | — |
| Go supervision (Part II component, B-001/B-010) | Boot-ladder + install-from-bundle error paths | UT-053–UT-054, UT-056–UT-060 | IT-017 | — |
| Shell modules (Part II component) | Pre-spawn trust check + window/nav/consumer policy | UT-055, UT-061–UT-068 | — | — |
| Security invariants 13–18 (Part II) | Window flags, socket hardening, CSP | UT-078 | IT-021, IT-022 | E2E-034 |
| `internal/desktoprelease` (Part II) | Per-arch inventory/policy + channel authority | UT-069–UT-071, UT-081 | — | — |

## Unit Tests

### `internal/update` — unified mechanism (Spec: Implementation Design / Core Interfaces)

- **UT-001** (happy): `CheckAll` on a `direct-binary` install with a newer release — returns `MultiState{Aggregate: available, App: nil}` with runtime versions populated.
- **UT-002** (happy): `CheckAll` with desktop app installed (app.json present, older version) — `App` non-nil, both tracks `available`.
- **UT-003** (boundary): release equal to current version — `up-to-date`, no offer; release lower — refused as downgrade on apply.
- **UT-004** (error): checksum catalog signature invalid (tampered sigstore bundle fixture) — check fails, `ApplyRuntime` never reachable, install untouched.
- **UT-005** (happy): `ApplyRuntime` on `desktop-app` install method — swaps `~/.compozy/bin/compozy` (S1 decision: no longer refused), backup created, `Finalize` removes it.
- **UT-006** (happy): `RequestAppApply` — acquires the Update Operation (or, for a dormant same-target record, executes the single authoritative **resume** transition per ADR-009), records the app section with verified asset identity + attempt id, and returns `accepted`; never a terminal verdict (B-006); a live-lease record yields `blocked`.
- **UT-007** (error): feed unreachable — background-mode check returns cached state with error recorded; forced check returns the failure.
- **UT-008** (state): `homebrew`/`npm`/`deb` methods — `managed: true`, recommendation string exact, `ApplyRuntime` refuses with managed error.
- **UT-009** (state): post-swap health failure — `Restore` puts the backup binary back byte-identical; state reports `failed` with restore message.
- **UT-010** (error): unsupported GOOS/arch and unwritable target dir — deterministic errors, no partial files on disk.
- **UT-011** (concurrency): acquisition against an existing record — a **live-leased** record returns `blocked` naming the holder; a **dormant** same-target record takes the single-winner resume transition; acquisition itself is lock-serialized create-if-absent with an fsynced temp record atomically published, so a concurrent reader never observes a partial record (B-021/B-025).
- **UT-012** (happy): provenance file rewritten inside the lock scope on desktop-app apply — `detect.go` re-classification still yields `desktop-app` afterward.
- **UT-013** (idempotency): `Finalize` twice and `Restore` after `Finalize` — second calls are safe no-ops with explicit results.
- **UT-014** (state): aggregate precedence table — every combination of per-target statuses maps to the documented aggregate (failed > blocked > staged > updated > available > up-to-date).
- **UT-015** (boundary): artifact-size policy at exactly 128 MiB archive / 256 MiB binary — pass at limit, `ArtifactSizeError` one byte over.
- **UT-016** (state): operation record with unknown `schema_version` — every reader (CLI, daemon, shell consumer, recovery sweep) refuses with a deterministic error and preserves the record for diagnosis.
- **UT-017** (happy): lockstep release lookup — app latest version derived from the same GitHub release as the runtime asset set.
- **UT-018** (ordering): `CheckAll` populates `App.Running` from the live app record (pid alive + start-time match), false for stale records.

### Update Operation + coordinator (Spec: ADR-009, Data Models, Release compatibility gate)

- **UT-074** (error): compatibility gate, correctly scoped — a **runtime-only** request against a release whose `min_app_version` exceeds the installed app is refused by the coordinator's backstop before any mutation; the **publication invariant** is tested at its own owner (the channel authority refuses publishing a release whose `min_app_version` exceeds the previous generation's app version); the default multi-target request against any published release succeeds runtime-first; no app installed → gate passes vacuously (B-004/B-020).
- **UT-075** (concurrency): parallel `O_EXCL` acquisition attempts (goroutine + subprocess table) — exactly one creates the record; every loser reads the same holder identity; no partial record ever exists on disk.
- **UT-076** (state): journal transition + recovery matrix — for every phase in the per-phase recovery/cancel matrix, the recovery function returns exactly the contracted action (resume / wait / restore / fail / cancel-allowed); every transition goes through the single Go API under the companion lock with operation_id + executor_generation + **expected revision** CAS — a stale writer (wrong revision or wrong generation) is refused with its evidence, never lost-update; terminal archive/removal is idempotent by operation id (B-012/B-021/B-022).
- **UT-083** (state): lease + dormancy semantics — a holder whose pid is dead, whose pid_start_time mismatches, or whose lease expired is not live; an expired-then-delayed executor's transition is refused by the generation fence **before any side effect**; `waiting-for-app` has a null holder, blocks nothing, and is resumable single-winner; cancel succeeds on dormant records, declines on live leases, is refused outright for the non-cancelable `installer-handoff` phase; cancel racing resume has exactly one winner (B-013/B-022).
- **UT-077** (error): app attempt digest binding — a downloaded app artifact whose digest differs from the operation's recorded identity is refused before install; the attempt journals `failed` with the mismatch evidence (B-006).
- **UT-080** (state): event coverage matrix — every journaled phase transition emits its canonical `update.*` event with all required correlation fields; the matrix test fails on any silent transition (N-002).

### Control-channel hardening (Spec: security invariants 17–18)

- **UT-078** (error): socket preconditions table — symlinked parent dir, wrong-owner dir, mode ≠ `0700`, missing/wrong/stale (pre-rotation) capability token (timing-safe comparison), over-cap message, and second in-flight request per connection are each refused deterministically; stale socket removed only after failed probe + ownership check (B-007/B-016 — no peer-credential branch exists, B-025).

### `compozy update` CLI (Spec: Agent Manageability Plan; `_dx.md` transcripts)

- **UT-019** (happy): `-o json` output for the running-app apply — byte-equal to the `_dx.md` golden transcript (both targets `updated`).
- **UT-020** (happy): no app installed — record has no `app` key at all (marshal asserted on raw JSON).
- **UT-021** (state): managed install — `recommendation` present, exit 0, nothing mutated.
- **UT-022** (error): blocked — aggregate `blocked`, holder pid in message, exit 1, record emitted before error.
- **UT-023** (happy): `--check` never mutates — filesystem snapshot identical before/after.
- **UT-024** (state): staged outcome — aggregate `staged`, app message names next-launch semantics, exit 0.
- **UT-025** (error): apply failure mid-daemon-restart — record `failed`, restore executed, message contains both causes.
- **UT-026** (happy): human and toon renderings carry the same ordered fields as the json record.

### `compozy app` preserved verbs (Spec: shell control subset)

- **UT-027** (happy): `app status` for installed+running, installed+stopped, and not-installed fixtures — schema-valid, exit 0 each.
- **UT-028** (error): app.json with an unknown `schema_version` (e.g. 99) — `app_state_schema_unknown` envelope, exit 1; a v1 record against the v2-validating CLI fails the same way (the designed evolution mechanism, B-023).
- **UT-029** (concurrency): status read racing a record rewrite — never a torn/partial decode (atomic rename contract).
- **UT-030** (error): `app open` argument validation table — relative, `..`, host-bearing, `\`, scheme-smuggling inputs each → `invalid_target_path`.
- **UT-031** (error): `app open` with no installation — `app_not_installed`.
- **UT-032** (error): dead socket file — `app_not_running` within the 2s default timeout.
- **UT-033** (happy/state): `app retry` healthy no-op leaves app.json byte-identical (regression: healthy-retry corruption bug).
- **UT-034** (error): `app diagnose --bundle` consent, unwritable output path, and mid-update states — each deterministic per `_dx.md`.

### Settings update surface (Spec: API Endpoints)

- **UT-035** (happy): `GetUpdate` maps `MultiState` to the extended payload; `app: null` and `operation: null` omitted-vs-present rules exact.
- **UT-036** (happy): `ApplyUpdate("runtime")` durably acquires the operation, spawns the coordinator **detached** (spawn recorded by fake), and returns `accepted` + operation id — it never executes the swap in-process (invariant 4).
- **UT-037** (happy): `ApplyUpdate("app")` returns `accepted` + operation id whether the shell is running or not; the resulting record phase is `staged` and the projection reports `staged` until the consumer transitions it.
- **UT-038** (error): unknown target → 400 envelope; nil updater service → 503 unavailable.
- **UT-039** (state): acquisition contention maps to the deterministic `blocked` body with the recorded holder.
- **UT-079** (happy): projection completeness — every field `_uiux.md` S1 renders (per-track status/versions/release_url/recommendation/restored_version/last_error + operation id/phase/percent/holder) is present field-for-field in the GET payload for a fixture of each state; nothing the UI needs is absent (B-005).
- **UT-082** (state): workspace-isolation evidence — the update surface exposes no workspace-scoped parameter, query key, or route variant, and projections are byte-identical across two workspaces on one home (Compozy Impact Audit).

### Web — S1 section + S2 indicator (Spec: `_uiux.md`; canonical suites `-general.test.tsx`, os-menubar)

- **UT-040** (happy): two-track render with both `available` — versions, pills, release links per track.
- **UT-041** (state): `app: null` payload → single-track section, no empty app row.
- **UT-042** (state): managed runtime — recommendation verbatim, apply affordance **absent from DOM**.
- **UT-043** (happy): apply click fires the mutation with target and renders staged-phase progress from subsequent payloads.
- **UT-044** (state): app `staged` — next-launch copy rendered.
- **UT-045** (state): daemon unreachable mid-apply — checking/unavailable view, then post-reconnect truth.
- **UT-046** (state): refresh failure with snapshot — last-known + refresh-failed pill + retry.
- **UT-047** (error): apply mutation `blocked` response — holder message surfaced, no optimistic success.
- **UT-048** (happy): keyboard — apply button reachable in tab order and Enter-activatable.
- **UT-049** (happy): indicator visible iff any track `available`.
- **UT-050** (state): indicator hidden for applying/staged/failed/up-to-date payloads.
- **UT-051** (happy): indicator activation navigates to the settings Updates section.
- **UT-052** (happy): indicator focus-visible state and Enter/Space activation.

### Runtime supervision policy (Go — relocated per B-001; Spec: Runtime supervision component)

- **UT-053** (happy): boot ladder resolver — attach chosen over start over provision given probe fixtures.
- **UT-056** (error): version probe below `MINIMUM_RUNTIME` / runtime `min_app_version` above the app — refusal with repair message in both directions (B-004 attach gate).
- **UT-057** (state): probe classification table — healthy, listening-unhealthy, stale-record, port-conflict fixtures.
- **UT-058** (state): stale daemon record cleanup decision.
- **UT-059** (state): supervision policy — backoff 500ms→10s, give-up after 5 unreadied starts, counter reset only on readiness.
- **UT-060** (concurrency): retry request during in-flight attempt — queued/refused, never parallel attempts.
- **UT-054** (error): install-from-bundle interrupted/disk-full — clean temp state, safe retry decision, never an executable partial binary (Go bootstrap owner, B-010).

### Shell main-process modules (TypeScript, colocated in `desktop/`)

- **UT-055** (error): pre-spawn bundle verification — the **shell** checks the bundled runtime payload digest against its build-embedded manifest before first spawn: missing or corrupt payload → refusal with reinstall-the-app guidance, nothing spawned, nothing installed (trust-root owner per B-027 — the payload never self-certifies).
- **UT-061** (state): window-state clamp — off-screen, disconnected-display, DPI-change fixtures land fully visible.
- **UT-062** (error): corrupt window-state JSON — defaults, healthy rewrite.
- **UT-063** (state): zoom clamp at bounds, step math, re-apply-on-reload value.
- **UT-064** (error): nav guard table — external http(s) → openExternal; mail/file/custom schemes → denied; in-origin → allowed.
- **UT-065** (state): crash-recovery budget — 3 reloads per rolling 60s, then dialog; unresponsive logged distinctly.
- **UT-066** (error): deep-link validation table mirrors UT-030 hostile inputs → default view.
- **UT-067** (state): deep-link queue — pre-ready queuing, exactly-once flush, last-wins for bursts.
- **UT-068** (state): app-track consumer recovery — `staged → applying` transition journaled before the installer runs, watchdog-deadline outcome on silent `quitAndInstall` failure, post-restart version-truth check gating `verified`, consecutive-failure escalation in the record (invariant 7).

### `internal/desktoprelease` + config consumers

- **UT-069** (happy/error): per-arch inventory exact-set assertion (ADR-008 names) — extra, missing, and empty-file fixtures each fail.
- **UT-070** (boundary): version strictly-greater policy at equal and pre-release boundaries.
- **UT-071** (error): cutover reference scan — repo grep fixture asserting zero live references to deleted Tauri surfaces (guards US-027/US-028.EC-1).
- **UT-072** (state): `[app]` key consumption — `update_check=false` disables both-track background checks; interval clamp at 15m/168h bounds errors exactly as today.
- **UT-073** (withdrawn): the shell no longer reads `[app]` keys — the daemon is the sole cadence consumer (B-015); no mirrored TypeScript validation exists.
- **UT-081** (happy/error): publish/repair authority — known-good selection from audit-marked commits; refusal when any rollback asset is missing from the versioned-release inventory; publish ordering places the ref CAS strictly after payload verification; a lost CAS yields deterministic `channel_cas_conflict`; re-running the same operation id converges idempotently instead of double-flipping (B-008/B-024).

## Integration Tests

### Daemon apply / restart / rollback (mock feed, real wiring)

- **IT-001**: `POST /api/settings/update/apply {target:runtime}` against mock feed — binary swapped, settings-restart operation reaches `ready`, `GET` reflects `updated`.
- **IT-002**: post-swap health probe forced to fail — automatic `Restore`, old binary byte-identical, status `failed` with restore evidence.
- **IT-003**: tampered artifact from mock feed — POST returns `accepted` + operation id (async contract); the coordinator's verify step journals `failed` pre-swap; GET then reports `failed` with the verification cause; install untouched (B-018 alignment).
- **IT-004**: `compozy update` (real CLI binary, mock feed, temp home) — full multi-target run matches `_dx.md` staged transcript with app.json fixture present, app closed.
- **IT-005**: kill daemon mid-apply — next boot resolves journal/backup to a working binary; status truthful.
- **IT-006**: apply with request context canceled immediately — operation continues (detachment), observable via operation id (SD-010 invariant 4).
- **IT-007**: `RequestAppApply` with a running-shell simulator — the consumer transitions `staged → applying → verified` inside the record, installs the recorded digest, and the GET projection reflects each phase; exactly one transition sequence occurs.
- **IT-008**: operation with app section written while the shell is absent — record persists; the simulated next-launch consumer resolves it; a second resolution attempt is a deterministic no-op (exactly-once, invariant 7).
- **IT-009**: operation record with future schema_version — daemon boot sweep, shell consumer, and CLI all refuse identically; the record is preserved.
- **IT-010**: concurrent CLI apply + daemon apply on one home — exactly one acquires the operation record (`O_EXCL`), the loser returns `blocked` with the lease holder identity (real processes; operation acquisition is the contended primitive, never a peer flock — B-018).
- **IT-011**: `blocked` surfaced identically through CLI record and HTTP body for the same contention.
- **IT-012**: HTTP and UDS `GET /api/settings/update` return identical payloads for identical state (transport parity).
- **IT-013**: HTTP and UDS `POST .../apply` parity including error bodies.
- **IT-014**: HTTP and UDS serve the extended GET payload and the apply route with identical observable behavior through each transport's full stack — route contract only; generated-artifact drift is owned solely by `make codegen-check` (already in the gate), and no test freezes generated output (N-004).
- **IT-015**: desktop-app provenance flow — Go apply rewrites provenance; subsequent `detect.go` classifies `desktop-app`; deleting provenance degrades to `direct-binary` classification.
- **IT-016**: `[app] update_check=false` in a real config home — daemon background checker never fires (observed over interval); manual endpoints still work.
- **IT-017**: `compozy app retry` against a live control-server fixture in failed state — boot re-driven; against healthy — `{ok:true}` no-op.
- **IT-018**: cross-process writer races — real coordinator, shell-consumer, daemon-sweep, and shell-sweep processes race **transitions** (not just acquisition) on one record: the companion lock + expected-phase CAS serialize them, no phase write is ever lost, app `applying` never overlaps a non-terminal runtime phase, and every loser reports `blocked`/stale-writer with the lease holder (B-003/B-012).
- **IT-019**: recovery end-to-end — for each non-terminal journaled phase fixture (`swapping`, `restarting`, `health-checking`, app `applying` past watchdog), daemon boot sweep and shell launch sweep resolve to the contract's action, ending in a working install and a terminal history record (B-002).
- **IT-020**: compatibility backstop under the operation — a **runtime-only** request against a hand-built release (publication invariant bypassed) with `min_app_version` above the installed app: the coordinator journals `failed` pre-mutation, the install is untouched, and CLI/GET report the compatibility explanation (B-004/B-020).
- **IT-021**: control-channel boundary — a request without the capability token, with a wrong token, and with a stale (pre-rotation) token are each refused deterministically (timing-safe comparison); connection and message bounds enforced against a live server; the token file is `0600` and rotated per shell start (B-007/B-016).
- **IT-022**: daemon-served CSP **protection** — responses carry exactly the production policy of security invariant 16 including `frame-ancestors 'none'`, and negative cases are enforced: an injected inline script does not execute, an unlisted connect origin is refused, an external page cannot frame the localhost UI, `unsafe-eval` is absent; dev-mode relaxations are proven absent from packaged serving (B-019/B-026).
- **IT-023**: compatibility-bump protocol — the channel authority **refuses to publish** a release whose `min_app_version` exceeds the previous generation's app version (publisher-gate case); a two-release sequence (N+1 app-compatible, N+2 requiring N+1's app) then flows through the frozen closed-app outcome at each step — runtime now, app staged — with no deadlock and no incompatible pair ever installed (B-020).
- **IT-024**: cancel parity — `compozy update --cancel` and `POST /api/settings/update/cancel` produce identical outcomes for dormant records (archived `canceled`) and identical declines for live leases, on both transports (B-013).

## End-to-End Tests

### Desktop lane (`make test-e2e-desktop`, Playwright `_electron`, real daemon, mock feed)

- **E2E-001**: first run on empty home — boot window shows resolve→provision→start→ready, main window lands on SPA, `compozy app status` reports owned runtime (US-001 AC-1..5).
- **E2E-002**: install-from-bundle failure — a corrupted bundled-runtime fixture boots to the failure phase with reinstall guidance and Retry; a healthy package then completes first run **with networking disabled** (offline-first-run proof); no partial binary ever existed (US-001.AC-6, EC-1/2).
- **E2E-003**: runtime binary fixture below minimum — refusal screen with repair path (US-003.EC-1).
- **E2E-004**: second launch (and launch burst) — single instance, window focused; with deep link → focused + navigated (US-007, US-013 AC-2).
- **E2E-005**: attach (daemon pre-started) and start (installed, stopped) journeys land in SPA without duplicate runtime processes (US-003/US-004).
- **E2E-006**: unhealthy-listener and never-ready fixtures — explain-the-failure dialog with retry/open-logs/quit; retry succeeds after fixture heal (US-004/US-005).
- **E2E-007**: quit while runtime serves CLI — process survives, relaunch attaches; quit during runtime apply waits quiesce (US-006).
- **E2E-008**: move/resize/maximize → relaunch restores; saved bounds forced off-screen → clamped visible (US-008).
- **E2E-009**: zoom in/out/reset via menu + shortcuts, persisted across relaunch and reload (US-009).
- **E2E-010**: external link click + `window.open` + hostile top-level nav — OS-browser spy receives only safe http(s); window stays on product (US-010).
- **E2E-011**: engine parity — the daemon-served Playwright journey suite runs green against the app shell (same specs, `_electron` fixture) including a glass/blur-bearing OS-shell screen (US-011).
- **E2E-012**: renderer kill — auto-reload within budget, sessions intact; forced crash-loop → dialog (US-012).
- **E2E-013**: `compozyos://open/sessions/<id>` running-app navigation; unknown path → SPA not-found (US-013).
- **E2E-014**: hostile deep links (`/http://evil.com`, `/../../etc`, `//host`) → default view every time (US-013.EC-1).
- **E2E-015**: cold-start deep link — app launches, boots, lands on target; hostile cold-start → default view (US-014).
- **E2E-016**: app update cycle via mock feed — indicator absent → feed bumps → menubar indicator appears, Settings apply → download/verify progress → restart → new version; strings match `_dx.md` overlay contract (US-015, US-029 AC-5).
- **E2E-017**: silent-install-failure fixture (updater exits without swap) — next boot reports old version truthfully, checks suppressed, retry succeeds (US-016).
- **E2E-018**: runtime update from Settings while sessions active — quiesce messaging, staged phases, health verify, version bump in status (US-017).

### Browser lane (extends `make test-e2e-web`, daemon-served SPA)

- **E2E-019**: Settings Updates section states — both-available, managed (no affordance), no-app single-track (US-029 AC-1/AC-3, EC-1, EC-3).
- **E2E-020**: apply runtime from a plain browser — progress renders, daemon restart reconnect handled truthfully, post-update truth (US-029 EC-2/EC-5).
- **E2E-021**: menubar indicator lifecycle in browser — appears on available, hidden while applying/failed, activation lands on section (US-029 AC-2, EC-4).
- **E2E-022**: keyboard-only journey — indicator → section → apply, no pointer (US-029 AC-4).

### CLI journeys (golden `_dx.md` transcripts, temp homes)

- **E2E-023**: `compozy update --check` / apply / staged / managed / blocked transcripts byte-compared (US-022).
- **E2E-024**: `compozy app status` (installed/running, not installed), `retry` healthy no-op, `diagnose` + `--bundle --yes` archive exists with manifest (US-020/US-023/US-024).
- **E2E-025**: `compozy app open` valid path (running + cold-start) and each invalid-path error (US-021, US-014.EC-2).
- **E2E-026**: `compozy update` offline (feed down) — silent background vs explicit manual failure shapes (US-015.EC-1, US-017.EC-3).
- **E2E-027**: headless host (no app.json, no app) — record without `app`, update applies, daemon restarted (US-018, US-022.EC-3).
- **E2E-028**: `compozy app update` → unknown-command error (US-022.AC-4).

### Pipeline & release (PIPE gates + rehearsal)

- **E2E-029**: packaged-app smoke (successor of `smoke-desktop-release-artifact.sh`) — isolated home, real packaged shell boots, provisions, reaches `/api/status`, teardown clean.
- **E2E-030**: per-arch mac manifest merge rehearsal — merged channel manifest routes arm64/x64 fixtures to correct assets (ADR-008).
- **E2E-031**: N→N+1 rehearsal against mock feed in CI — packaged app at N auto-updates to N+1 and relaunches (automated stand-in for the QA-recorded real cycle).
- **E2E-032**: publish-authority ordering — `compozy-desktop-release publish` dry-run plus an interrupted rehearsal prove payload upload + inventory verification precede the channel-head flip, and an interrupt before the flip leaves the previous generation fully live (behavior-level authority evidence; no workflow-file literals — B-018/N-004 discipline).
- **E2E-033**: repair rehearsal against the **real provider read path** — publish a known-bad generation to a staging channel branch, run `compozy-desktop-release repair`, and assert the generic-provider raw URLs serve the last known-good manifest set after the ref CAS, with the audit commit recorded; a repair with a missing rollback asset refuses; an interrupted publish at each step leaves the branch on exactly one complete generation (US-026, B-024).
- **E2E-034**: packaged-build security assertions — devtools cannot open under any env combination, a permission-requesting page is denied, a non-web scheme link never executes, the SPA loads under the production CSP with zero violations, an injected inline script + an unlisted connect attempt are both blocked in the real packaged window, and the **boot window's meta CSP** blocks the same negative cases (security invariants 13–16 driveable via the `_electron` fixture; B-019/B-026).
