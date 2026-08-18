# Test Specification: CompozyOS Desktop App

Canonical test contract for the CompozyOS Desktop App. Companion to `_techspec.md`. Derived from `_user_stories.md` (behavior) and `_techspec.md` (components). IDs are permanent once tasks reference them; dropped cases are marked `(withdrawn)` in place.

## Strategy

- **Frameworks and harnesses:** Rust `cargo test` for the shell (unit + integration; async via tokio test where needed) — IO behind traits (`RuntimeResolver`, artifact fetcher, process spawner) with in-memory/stub fakes at those boundaries only; temp-dir `COMPOZY_HOME`s per test. Go tests follow the repo's conventions (`t.Run("Should …")` subtests, `t.Parallel` default, table-driven, `-race`) in the owning packages' canonical suites (`internal/cli`, `internal/config`, `internal/update`). Shell integration tests use three shared fixtures: a **stub daemon HTTP server** (canned `/api/status`, `/api/settings/update`), a **fake daemon process** (tiny script honoring SIGTERM + writing `daemon.json`), and a **local updater fixture feed** (staging minisign keypair — never the production key — plus dummy artifacts and `latest.json`/`runtime.json`).
- **Execution:** `make desktop-test` (cargo fmt+clippy+test) in the desktop CI lane; Go cases run in `make test` / `make gate`; E2E journeys run against a real shell build + real daemon binary under isolated `COMPOZY_HOME`s — scripted via `tauri-driver` where the platform supports it (Linux/Windows; no macOS WebDriver), with macOS coverage landing in the per-release platform smoke matrix and QA charters. Release-integrity cases run in the release workflow's dry-run/draft-release path.
- **Conventions:** every unit case tagged (`happy|error|boundary|concurrency|idempotency|ordering|state`); one observable behavior per unit case; E2E cases are journeys through public surfaces (installer, window, CLI, feeds) exactly as a user or agent would drive them. No `t.Parallel` on env-mutating Go tests; no forced Playwright-style actionability bypasses.

## Coverage Matrix

| Source | Behavior | Unit | Integration | E2E |
| --- | --- | --- | --- | --- |
| US-001 | Install → working product, no prior setup | UT-029, UT-034, UT-038 | IT-003 | E2E-001 |
| US-001.EC-1 | Offline first run → honest retry | UT-031 | IT-012 | E2E-002 |
| US-001.EC-2 | Unwritable target named + retry | UT-033 | — | — |
| US-001.EC-3 | Disk-space failure, clean retry | UT-032 | — | — |
| US-001.EC-4 | Reinstall over existing → replaced in place | — | — | E2E-020 |
| US-001.EC-5 | Wrong-platform build refuses clearly | UT-038 | — | — |
| US-002 | Provisioning failure recovery | UT-035 | IT-011 | — |
| US-002.EC-1 | Launch race during first run → one flow | — | IT-020 | E2E-007 |
| US-002.EC-2 | Transient network retry, no reinstall | UT-031 | IT-012 | — |
| US-002.EC-3 | External runtime appears → attach, not duplicate | UT-036 | — | — |
| US-003 | Attach to running runtime, touch nothing | UT-013 | IT-001 | E2E-003 |
| US-003.AC-3 | Isolated home resolution | UT-010, UT-011 | IT-027 | — |
| US-003.EC-1 | Stale record → treated absent | UT-014, UT-015 | IT-026 | — |
| US-003.EC-2 | Foreign process on address → refused + named | UT-018 | IT-004 | — |
| US-003.EC-3 | Unhealthy runtime → honest degraded state | UT-023 | IT-028 | — |
| US-004 | Start installed-but-stopped runtime | UT-024 | IT-002 | E2E-004 |
| US-004.EC-1 | Start failure → evidence + retry, no loop | UT-025 | IT-008 | — |
| US-004.EC-2 | Slow start → bounded honest progress | UT-025 | — | E2E-011 |
| US-004.EC-3 | Simultaneous starters → exactly one runtime | UT-027 | IT-024 | — |
| US-005 | Version skew → guided state | UT-020 | IT-005 | E2E-005 |
| US-005.EC-1 | Runtime newer than app → guided (no forward-compat) | UT-021 | IT-006 | — |
| US-005.EC-2 | Version info unavailable → treated incompatible | UT-022 | — | — |
| US-006 | Unreachable states honest + actionable | UT-009 | IT-004 | E2E-011 |
| US-006.EC-1 | Repeated failures escalate detail (log tail) | UT-087 | — | — |
| US-006.EC-2 | External fix → retry succeeds without restart | — | IT-028 | — |
| US-007 | Runtime dies → graceful degrade + reconnect | UT-006, UT-008 | IT-007 | E2E-006 |
| US-007.EC-1 | Crash mid-action → UI error handling owns it | — | IT-007 | — |
| US-007.EC-2 | Rapid crash cycles → bounded, terminal state | UT-007 | IT-008 | — |
| US-007.EC-3 | Connection blip (runtime alive) → reconnect only | — | IT-007 | — |
| US-008 | Quit never stops runtime or agent work | UT-026 | IT-009 | E2E-004 |
| US-008.EC-1 | OS shutdown/logout | — | — | platform smoke charter (not CI-automatable; verified per release) |
| US-008.EC-2 | Force-kill app → no damage, next launch attaches | — | IT-010 | — |
| US-009 | Second launch focuses window | — | IT-020 | E2E-007 |
| US-009.EC-1 | Second launch with link → forwarded, not dropped | UT-048 | IT-020 | E2E-008 |
| US-009.EC-2 | Stale single-instance state after crash → recovers | UT-082 | IT-010 | — |
| US-010 | CompozyOS links open the right view | UT-044 | IT-019 | E2E-008 |
| US-010.AC-2 | Cold-start link survives | UT-048 | — | E2E-023 |
| US-010.EC-1 | Nonexistent entity → product not-found view | — | — | E2E-008 |
| US-010.EC-2 | Malformed/hostile payload → safe default view | UT-045, UT-046 | — | E2E-008 |
| US-010.EC-3 | Rapid links → last wins | UT-047 | — | — |
| US-010.EC-4 | Link during error state → preserved until ready | UT-048 | — | — |
| US-011 | External links → OS browser | UT-050 | — | E2E-009 |
| US-011.AC-2 | window.open external → OS browser | UT-051 | — | E2E-009 |
| US-011.EC-1 | Internal-looking foreign origin → external | UT-050 | — | — |
| US-012 | Geometry persists + invalid-state recovery | UT-052, UT-053 | — | E2E-010 |
| US-012.EC-1 | Ended minimized → opens visible | UT-054 | — | — |
| US-012.EC-2 | Display change → bounds validated | UT-053 | — | E2E-010 |
| US-013 | Flash-free bounded launch | UT-055 | — | E2E-011 |
| US-013.EC-1 | Slow machine → alive loading, bound applies | UT-055 | — | — |
| US-013.EC-2 | Launch into error → loading hands off cleanly | UT-009 | — | E2E-011 |
| US-014 | Background update → consent restart | UT-056, UT-058 | IT-013 | E2E-012 |
| US-014.AC-3 | Unverifiable artifact never applied | UT-057 | IT-014 | — |
| US-014.EC-1 | No network → silent skip, retried | UT-059 | — | — |
| US-014.EC-2 | Interrupted download → partial never applied | UT-060 | — | — |
| US-014.EC-3 | Quit with pending update → applied next launch | — | — | E2E-022 |
| US-014.EC-4 | Repeated failure → escalates to fallback | UT-061 | — | — |
| US-014.EC-5 | Sleep/wake across cycle | — | — | platform smoke charter (no reliable CI sleep simulation) |
| US-015 | Failed update never strands silently | UT-061, UT-063, UT-114, UT-115 | IT-015 | E2E-013 |
| US-015.EC-1 | Crash-on-new-version detectable + fallback reachable | — | — | E2E-013 |
| US-015.EC-2 | Feed older than installed → ignored | UT-062 | — | — |
| US-016 | Provisioned runtime: one update experience | UT-067, UT-108 | IT-016, IT-029, IT-030 | E2E-014, E2E-025 |
| US-016.EC-1 | Runtime apply fails → previous usable + diagnostics | UT-068, UT-101 | IT-017 | — |
| US-016.EC-2 | Repeated decline → no nagging, offer stays | UT-070 | — | — |
| US-016.EC-3 | Ownership changed to managed → switch behavior | UT-069 | — | — |
| US-017 | Managed installs: recommendation, never a write | UT-065 | IT-018 | E2E-015 |
| US-017.AC-2 | Channel update reflected, no residual pending | — | IT-018 | E2E-015 |
| US-017.EC-1 | Inconclusive detection → safe recommendation + why | UT-066, UT-080, UT-081 | — | — |
| US-018 | Channel visibility (beta) | — | — | E2E-016 |
| US-018.AC-2 | No inactive stable choice offered | — | — | E2E-016 |
| US-018.EC-1 | Future graduation path | — | — | no v1 behavior (deferred decision per PRD; nothing to test) |
| US-019 | CLI open/status/update/retry/diagnose structured + deterministic | UT-071–UT-076, UT-109–UT-111 | IT-021, IT-031 | E2E-017, E2E-024 |
| US-019.EC-1 | `open` with target path parity | UT-074, UT-075 | — | E2E-017 |
| US-019.EC-2 | Transitional states reported truthfully | UT-043, UT-076 | — | E2E-017 |
| US-020 | Browser and app: same product, same state | — | — | E2E-003, E2E-018 |
| US-020.EC-1 | Different homes → each truthful, never mixed | — | IT-027 | — |
| US-021 | Release fails closed | UT-083, UT-085 | — | E2E-019 |
| US-021.AC-2 | Published feed verified end-to-end | UT-083, UT-084 | — | E2E-019 |
| US-021.AC-3 | Key custody runbook exists + reviewed | — | — | human release gate (checklist item, not automatable) |
| US-021.EC-1 | Partial platform failure → no silent drop | UT-083 | — | E2E-019 |
| US-021.EC-2 | Reserved stable path publication blocked | UT-086 | — | — |
| ShellState machine (TechSpec: Core Interfaces) | Transitions + error mapping | UT-001–UT-009 | — | — |
| home.rs | Home resolution rules (match-Go) | UT-010–UT-012 | IT-027 | — |
| runtime/discovery.rs | daemon.json + liveness | UT-013–UT-016 | IT-026 | — |
| runtime/probe.rs | **Bound** identity + handshake | UT-017–UT-023, UT-089–UT-091 | IT-004–IT-006 | — |
| runtime/resolver.rs + platform registration | Install resolvers + precedence | UT-105–UT-107 | — | — |
| runtime/supervisor.rs | Spawn/readiness/quit/stop-guard | UT-024–UT-028 | IT-002, IT-009 | — |
| runtime/mutation.rs | Update lock + journal + crash recovery | UT-092–UT-095, UT-100–UT-102 | IT-016, IT-029 | — |
| runtime/quiesce.rs | Drain contract client + revalidation | UT-067, UT-108 | IT-030, IT-032 | — |
| runtime/provision.rs + artifacts.rs | Signed manifest + pipeline + marker + atomicity | UT-029–UT-038, UT-096–UT-098 | IT-003, IT-011 | — |
| errors.rs redaction gate | Typed errors + secret containment | UT-099, UT-103, UT-104 | — | — |
| control.rs + `app.sock` | Control protocol + CLI parity | UT-109–UT-111 | IT-031 | E2E-024 |
| desktop/schema corpus | Cross-language schema gate | UT-042, UT-104 | IT-021, IT-022 | — |
| config.rs (Rust) | `[app]` read + defaults + fail-closed | UT-039–UT-041 | IT-022, IT-025 | — |
| record.rs | app.json atomic writes | UT-042, UT-043, UT-088, UT-099 | IT-021 | — |
| Brand sweep gate (ADR-012) | No product-language "Compozy" outside allowlist | UT-113 | — | — |
| links.rs | Validation + queue | UT-044–UT-048 | IT-019 | — |
| nav.rs | Fencing + external handoff | UT-049–UT-051 | — | E2E-009 |
| windowing.rs | Geometry + flash-free bounds | UT-052–UT-055 | — | E2E-010, E2E-011 |
| update/app_update.rs | Updater orchestration + watchdog + hardening | UT-056–UT-064, UT-114, UT-115 | IT-013–IT-015 | E2E-012 |
| update/runtime_update.rs | Ownership split + apply | UT-065–UT-070 | IT-016–IT-018 | E2E-014 |
| internal/cli app verbs (Go) | Status/open + errors | UT-071–UT-076, UT-082 | IT-021 | E2E-017 |
| internal/config AppConfig (Go) | Lifecycle | UT-077, UT-078 | IT-022 | — |
| internal/update detection (Go) | desktop-app method + recommendation mapper | UT-079–UT-081, UT-112 | IT-023 | — |
| Release pipeline gates | Feed/signing validation (incl. signed runtime manifest, BR-10 gates) | UT-083–UT-086, UT-113, UT-116 | — | E2E-019 |
| SSE under OS webviews (PRD constraint) | Streaming load validation | — | — | E2E-021 (release gate) |

## Unit Tests

### ShellState machine (TechSpec: Core Interfaces — `state.rs`)

- **UT-001** (happy): `ShellState::advance` — given `Resolving` and `Resolution::Attached{origin, owned:false}`, produces `Handshaking` then `Product{origin, owned:false}` on handshake-ok event.
- **UT-002** (state): given `Resolving` and `Resolution::NeedsProvision`, produces `Provisioning{stage: Download{pct:0}}`.
- **UT-003** (state): given `StartingRuntime{attempt:1}` and readiness-ok event, produces `Handshaking`.
- **UT-004** (error): given `Handshaking` and probe result `version < min supported`, produces `Skew{newer:false}` carrying both versions.
- **UT-005** (boundary): given probe result `StatusSchemaVersion = max_known + 1`, produces `Skew{newer:true}`.
- **UT-006** (state): given `Product` and health-loss event, produces `Disconnected{cause: RuntimeDown}`.
- **UT-007** (boundary): given `StartingRuntime{attempt:3}` and start-failure event, produces `ShellError{code: RuntimeStartFailed}` — never `StartingRuntime{attempt:4}`.
- **UT-008** (state): given `Disconnected` and probe-ok event, produces `Product` without app restart.
- **UT-009** (error): every `ShellErrorCode` variant maps to a `ShellError` state with non-empty `detail` and a `log_path` — exhaustive match test (compile-time + runtime assertion).

### Home resolution (`home.rs`)

- **UT-010** (happy): `resolve_home` — with `COMPOZY_HOME=/tmp/x` set, returns `/tmp/x` (no hardcoded fallback).
- **UT-011** (happy): with the variable unset, returns the platform default home identical to the runtime's rule (`~/.compozy`).
- **UT-012** (state): Rust home resolution matches Go home resolution byte-for-byte — for relative, `~`-prefixed, whitespace, unset, and invalid-user-home inputs, both produce the same **absolute** result (the runtime absolute-izes a relative `COMPOZY_HOME`; it does not reject). Asserted against the shared cross-language fixture table; rewritten per peer review B-009 — the earlier "reject relative" expectation encoded false semantics.

### Daemon discovery (`runtime/discovery.rs`)

- **UT-013** (happy): valid `daemon.json` `{pid, port, started_at}` + matching live process (fake PID table) → `Live{port}`.
- **UT-014** (error): record whose PID is dead → `Absent` (never `Live`) — BR-5.
- **UT-015** (error): PID alive but process start-time mismatches `started_at` → `Absent` (PID-reuse defense).
- **UT-016** (error): malformed/truncated `daemon.json` → `Absent` with a parse diagnostic (no panic).

### Identity probe + handshake (`runtime/probe.rs`)

- **UT-017** (happy): stub `/api/status` returning the real contract shape (date-string `schema_version`, nested `daemon.version/pid/started_at/http_host/http_port`) whose PID, start time, and port all agree with the validated `daemon.json` record → `Identity::Compozy(bound)`. Shape alone is never sufficient (B-002).
- **UT-018** (error): responder returning HTML or a foreign JSON shape → `Identity::Foreign` → `ShellErrorCode::PortConflictForeign`.
- **UT-019** (boundary): daemon version exactly equal to the pinned minimum → compatible.
- **UT-020** (error): daemon version below minimum → skew-older with the runtime-channel recommendation string.
- **UT-021** (error): `schema_version` date-string unknown/newer than the app's known constant → skew-newer with "update the app" recommendation.
- **UT-022** (error): status payload missing version fields → treated as incompatible (skew state), not a crash.
- **UT-023** (error): probe exceeding its request deadline → `RuntimeUnhealthy` (deadline is asserted, no unbounded wait).
- **UT-089** (error): well-formed payload whose `daemon.pid` differs from the record's PID → `Identity::Foreign` (copied-payload squatter defense, B-002).
- **UT-090** (error): PID agrees but start time or port disagrees with the record → `Identity::Foreign` (PID-reuse/replay defense).
- **UT-091** (state): shell-started daemon — payload PID must also equal the spawned child PID; mismatch → `Identity::Foreign`.

### Supervisor (`runtime/supervisor.rs`)

- **UT-024** (happy): `start_owned` spawns detached (fake spawner), polls readiness until discovery+probe pass → returns `Owned{pid}`.
- **UT-025** (error): readiness deadline (30s, injected clock) elapses → `RuntimeStartFailed` carrying the captured stderr/log tail; exactly the bounded number of poll attempts occurred.
- **UT-026** (state): `on_quit` with an owned running daemon → no signal sent, daemon record untouched (BR-2 — assert the fake process received nothing).
- **UT-027** (concurrency): two concurrent `start_owned` calls (simulated CLI race) → exactly one spawn wins; loser resolves to attach against the winner's discovery record.
- **UT-028** (state): `stop_for_update` guard — permitted only when durable ownership verifies (provenance marker + executable path/hash agreement, B-003); refuses attached/unowned daemons with `NotOwned` (BR-1). A relaunched shell CAN stop a surviving app-provisioned daemon it did not spawn this process lifetime.
- **UT-092** (concurrency): `update.lock` contention — second mutator (simulated `compozy daemon start`) blocks or fails cleanly while the transaction holds the lock; no interleaved mutation (B-003).
- **UT-093** (error): stale `update.lock` (dead PID / start-time mismatch) → detected and reclaimed; live holder never displaced.
- **UT-094** (state): after lock acquisition the runtime state is re-probed — a daemon that appeared meanwhile switches the flow to attach, not mutation.
- **UT-095** (error): provenance marker present but `binary_sha256` disagrees with the on-disk binary → ownership refused → recommendation-only (BR-8).

### Provisioning pipeline (`runtime/provision.rs`, `runtime/artifacts.rs`)

- **UT-029** (happy): signed `runtime.json` — minisign signature verifies over canonical bytes, then the current platform's `{url, sha256}` entry is selected (B-001).
- **UT-096** (error): manifest with invalid/absent `.minisig` → hard refusal before any field is trusted; no download occurs.
- **UT-097** (error): manifest signed by a different key → refusal (pinned pubkey only).
- **UT-098** (error): manifest missing `schema_heads` → refusal (migration preflight impossible, B-005).
- **UT-030** (error): downloaded archive hash ≠ manifest `sha256` → `VerifyFailed`; staged file deleted; existing binary and marker byte-identical to before.
- **UT-031** (error): fetch error mid-download → `ProvisionNetwork` with resumable state; retry re-fetches without requiring app reinstall.
- **UT-032** (error): simulated ENOSPC during staging → `ProvisionDiskSpace` naming the required size; staging dir cleaned.
- **UT-033** (error): unwritable `$COMPOZY_HOME/bin` → `ProvisionPermission` naming the exact path.
- **UT-034** (happy): successful pipeline → binary installed via stage-then-rename; `.desktop-provenance.json` written with `installed_by:"desktop-app"`, matching `binary_sha256`, channel, versions.
- **UT-035** (state): interrupted install (staged file present, no marker) detected at next launch → `Provisioning` resumes with continue/start-over choice (US-002.AC-2), and both paths converge to UT-034's postcondition.
- **UT-036** (state): discovery finds a live runtime while in provisioning-error state → provisioning aborts, resolution switches to attach (US-002.EC-3).
- **UT-037** (boundary): manifest `version` ≤ installed runtime version → no update offered (downgrade/no-op guard, BR-10).
- **UT-038** (error): manifest lacking an entry for the current target triple → clear unsupported-platform refusal (US-001.EC-5).

### Shell config (`config.rs`)

- **UT-039** (happy): missing file or missing `[app]` section → defaults `{update_check: true, update_check_interval: 6h}`.
- **UT-040** (error): present-but-invalid section (malformed TOML, unknown key, bad duration) → update consumer **fails closed**: no check scheduled, `ConfigInvalid` diagnostic in `app.json`; shell startup itself proceeds (ADR-010 amended, B-011 — never "default through" rejected config).
- **UT-041** (boundary): interval below 15m or above 168h → rejected exactly as the Go validation rejects it (shared validation corpus, same fixtures as UT-078) → fail-closed path of UT-040.

### App record (`record.rs`)

- **UT-042** (happy): transition write produces `app.json` that validates against `desktop/schema/app-state.schema.json` (schema_version 1, atomic temp+rename — no partially-written reads under concurrent reader loop).
- **UT-043** (state): `Provisioning{Verify}`, `Attaching`, `Updating{Runtime}`, and `update.app_state=downloading` are written verbatim — the full canonical vocabulary is representable and transitional states visible (US-019.EC-2, B-007).
- **UT-088** (error): write failure (read-only dir) → logged, shell continues (record is observability, never a crash cause).
- **UT-099** (error): `state=error` writes carry the typed `{code, safe_message, log_path}` error object — never a bare string (B-007/B-012).

### Update journal + crash recovery (`runtime/mutation.rs`)

- **UT-100** (state): each phase transition (`staged→stopped→swapped→started→done`) is fsync'd before the action it precedes; the journal sequence for a clean update is exactly this order.
- **UT-101** (error): crash simulated at every phase boundary → recovery classifies correctly: pre-`swapped` → clean rollback with old binary intact; post-`swapped` pre-`started` → staged binary completes or rolls back per journal; post-`started` boot failure → `recovery_required`, old binary NOT restarted (B-003/B-005).
- **UT-102** (state): `recovery_required` is sticky across shell relaunches until resolved by a later signed build or explicit operator action; `app.json.update.runtime_state` reports it.

### Redaction gate (`errors.rs`)

- **UT-103** (error): adversarial corpus — inputs containing `compozy_claim_*`, `claim_token`, bearer tokens, `vault:` refs, control characters, and >512-byte bodies pass through every public sink path (`app.json`, log lines, control-socket responses, CLI output fixtures) with secrets absent and length bounded (B-012).
- **UT-104** (state): `ShellError` construction from an HTTP failure never embeds the response body — only the typed code + producer-authored message; the body goes to the log file alone.

### Install resolvers (`runtime/resolver.rs` + Go CLI side)

- **UT-105** (happy): runtime binary precedence — app-owned (provenance-valid) chosen over PATH install; live `daemon.json` process always attach-only.
- **UT-106** (error): GUI-launch environment (no shell PATH) → operator-install resolution still deterministic per platform (login-shell PATH resolution fixture); none found → `NeedsProvision`, never a guess (B-008).
- **UT-107** (happy, Go): app installation truth per platform fixture — macOS bundle-identifier query, Windows uninstall-key, Linux `.desktop` entry → `installed`/`app_version` derived from registration; `app.json` absence/staleness never flips `installed` (B-008).

### Control socket (`control.rs` + Go client)

- **UT-109** (happy): `update.check`, `update.apply(app|runtime)`, `retry`, `diagnose`, `navigate` round-trip over a temp socket with versioned envelope; responses schema-validated.
- **UT-110** (error): unknown verb / unknown protocol major → deterministic error envelope; socket created 0600 (permission asserted).
- **UT-111** (error, Go): `compozy app update --apply runtime` with no socket → `app_not_running`; socket present but unresponsive → `app_control_unavailable` (B-006).

### Deep links (`links.rs`)

- **UT-044** (happy): `compozyos://open/sessions/abc?tab=logs` → validated `ProductPath("/sessions/abc?tab=logs")`.
- **UT-045** (error): payload embedding a scheme or authority (`compozyos://open/http://evil.com`, `//evil.com/x`) → rejected → `DefaultView` (BR-16).
- **UT-046** (error): traversal payload (`/../etc`, encoded variants) → rejected → `DefaultView`.
- **UT-047** (ordering): two validated links delivered in quick succession → queue holds only the last (US-010.EC-3).
- **UT-048** (state): link arriving during `Provisioning`/`ShellError` → queued; on transition to `Product`, exactly one navigation to the queued path fires (US-010.AC-2/EC-4).

### Navigation fencing (`nav.rs`)

- **UT-049** (happy): navigation to daemon origin and to bundled page origin → allowed.
- **UT-050** (error): navigation to any other origin (including an internal-looking `http://localhost:9999`) → cancelled + opener invoked with that URL (US-011.EC-1: different port = different origin = external).
- **UT-051** (error): `window.open`/`target=_blank` to external origin → no new webview window; opener invoked (US-011.AC-2).

### Windowing (`windowing.rs`)

- **UT-052** (happy): saved geometry within current work areas → restored exactly.
- **UT-053** (error): geometry off-screen / below minimum / on a removed monitor (fixture display sets) → centered sane default (US-012.AC-2/EC-2).
- **UT-054** (state): previous session ended minimized → restore opens visible (US-012.EC-1).
- **UT-055** (boundary): first-paint signal absent at the 10s bound (injected clock) → transition to error state; signal at 9.9s → product shown (US-013).

### App updater orchestration (`update/app_update.rs`)

- **UT-056** (happy): fixture check returns newer version → download progresses → state `ready` + `app.json.update.app_available` set (US-014.AC-1).
- **UT-057** (error): artifact failing signature verification → never applied; state `failed` with diagnostic (US-014.AC-3).
- **UT-058** (happy): feed version equal to current → silent no-op, no state change beyond `last_checked_at` (US-014.AC-4).
- **UT-059** (error): check network failure → silent skip; `last_error` recorded; no dialog event emitted (US-014.EC-1, US-015.AC-2).
- **UT-060** (state): interrupted download → next cycle restarts/resumes; partial artifact never reaches apply (US-014.EC-2).
- **UT-061** (boundary): N consecutive failed cycles → escalation flag set → manual-download fallback surfaced (US-014.EC-4 → US-015.AC-1).
- **UT-062** (boundary): feed advertising an older version → ignored, no offer (US-015.EC-2, BR-10).
- **UT-063** (state): restart watchdog — app fails to exit within its bound after consent → fallback state with manual-download affordance (US-015.AC-1).
- **UT-064** (state): `update_check=false` → no check scheduled; `compozy app status` shows `app_state: idle` with no `last_checked_at` movement (ADR-010 off switch).
- **UT-114** (error): macOS apply hardening — the shell's own `.app` backup is taken before `install()` and restored when install fails (simulated failure) → previous app intact and launchable; the plugin's internal backup is never relied on (upstream #3505, analysis 06 §4).
- **UT-115** (error): feed endpoint returning 404/`ReleaseNotFound` → surfaced as a feed-error state (`update.app_state=failed` + `last_error` + `last_checked_at`), never conflated with the "up to date" no-op (analysis 07 §4; US-015.AC-2).

### Runtime updater (`update/runtime_update.rs`)

- **UT-065** (happy): `/api/settings/update` stub reporting `managed:true, install_method:"homebrew"` → recommendation surfaced verbatim; apply affordance absent; zero writes (US-017.AC-1).
- **UT-066** (error): provenance inconclusive (marker mismatch signaled by detection) → recommendation-only with the stated reason (US-017.EC-1, BR-8).
- **UT-067** (state): apply flow drives the daemon's real quiesce contract (drain → settle → safe-to-stop readout): while the readout reports unsettled work, the consent gate blocks; consent + safe-to-stop → proceed; declined → undrain, no stop (US-016.AC-2, BR-9, B-004 — no invented busy signal; the fixture implements the actual drain surface shapes).
- **UT-108** (concurrency): quiesce revalidation race — safe-to-stop turns false between readout and SIGTERM (new claim admitted in the window) → stop aborts, undrain restores admission, state returns to consent (B-004).
- **UT-068** (error): apply pipeline failure after stop (bad archive) → old binary intact, daemon restarted on old version, error state with diagnostics (US-016.EC-1).
- **UT-069** (state): ownership flip detected (method no longer `desktop-app`) → behavior switches to recommendation-only (US-016.EC-3).
- **UT-070** (state): decline recorded → offer remains visible in update surface; no re-prompt event within the same session (US-016.EC-2).

### CLI `compozy app` (Go — `internal/cli`)

- **UT-071** (happy): `compozy app status -o json` against a fixture `app.json` (state `product`, live PID) → golden `AppStatusReport` validating against the canonical schema, `installed` from platform-registration fixture (not from `app.json`), `running:true`, runtime attach fields.
- **UT-072** (happy): no platform registration found → `{installed:false, running:false}` exit 0 (status is truthful, not an error).
- **UT-073** (error): `compozy app open` with no installation → exit non-zero, `-o json` carries `error.code = "app_not_installed"` (US-019.AC-3).
- **UT-074** (error): `compozy app open '../evil'` and `compozy app open 'http://evil.com'` → `error.code = "invalid_target_path"` (validation parity with UT-045/046).
- **UT-075** (happy): `compozy app open /sessions/abc` → composed scheme URL `compozyos://open/sessions/abc` handed to the platform opener (fake opener records it).
- **UT-076** (state): fixture `app.json` in `provisioning`/`updating` states → status reports the transitional state string verbatim (US-019.EC-2).
- **UT-082** (state): fixture `app.json` with dead PID → `running:false` despite record presence (US-009.EC-2 status half; liveness algorithm shared with daemon status).

### Config lifecycle (Go — `internal/config`, canonical config suite)

- **UT-077** (happy): `DefaultWithHome` carries `App: {UpdateCheck: true, UpdateCheckInterval: 6h}`; TOML round-trip preserves values.
- **UT-078** (error): `update_check_interval = "5m"` (below floor) and `"200h"` (above ceiling) → validation error naming `app.update_check_interval`; same fixture table UT-041 mirrors.

### Install-method detection (Go — `internal/update`, existing detect matrix suite)

- **UT-079** (happy): executable at `<home>/bin/compozy` + marker with matching `binary_sha256` → `InstallMethodDesktopApp`, `managed:true`, recommendation naming the CompozyOS app.
- **UT-080** (error): marker present but `binary_sha256` mismatch → falls through to generic heuristics (inconclusive ⇒ not app-owned).
- **UT-081** (error): marker unreadable/corrupt JSON → falls through without error propagation (detection never fails the status call).
- **UT-112** (happy): the recommendation/state mapper renders the `desktop-app` method end-to-end in the existing recommendation matrix suite — `GET /api/settings/update` fixture shows `install_method:"desktop-app"`, `managed:true`, and the app-naming recommendation string (N-005).

### Release pipeline gates (feed/signing validation logic)

- **UT-083** (error): `latest.json` missing one of the four platform entries → validation fails the release step (US-021.EC-1).
- **UT-084** (error): `runtime.json` entry missing `sha256`, missing `schema_heads`, or malformed version → validation fails.
- **UT-085** (error): updater artifact present without its `.sig`, or `runtime.json` without a verifying `.minisig` (or signing env absent) → hard failure before publish (US-021.AC-1, B-001).
- **UT-086** (error): publish target resolving to `desktop/stable/` while stable is locked → refused (US-021.EC-2, ADR-005).
- **UT-113** (error): brand-sweep occurrence gate — product-language "Compozy" occurrences (outside the codified command-identifier allowlist) found in swept surfaces → gate fails (ADR-012; runs with the rename tasks and stays as a regression gate).
- **UT-116** (error): BR-10 regression gates — (a) the shipped shell binary uses the default strictly-greater comparator (no `allowDowngrades`, no custom `version_comparator`); (b) a release whose version is not strictly greater than the live feed's version fails the publish step (analysis 07 §7 gates 11–12).

### Error-state escalation

- **UT-087** (state): repeated retry from the same `ShellError` (3rd attempt, injected counter) → state detail escalates to include the runtime log tail instead of repeating the same message (US-006.EC-1).

## Integration Tests

### Resolution ladder (stub daemon + fake process + temp homes)

- **IT-001**: healthy stub daemon + valid `daemon.json` → shell resolves `Attached{owned:false}`; zero spawn calls; window target = stub origin.
- **IT-002**: installed fake-daemon binary, no live process → supervisor spawns it, readiness poll passes when it writes `daemon.json` + serves status → `Attached{owned:true}`.
- **IT-003**: empty home + fixture artifact server → full provisioning pipeline → binary + provenance marker on disk → fake daemon started → `Product` state; `app.json` recorded every stage in order.
- **IT-004**: foreign HTTP server squatting the port → probe classifies `Foreign`; state = `PortConflictForeign`; no navigation to the squatter (US-003.EC-2, US-006.AC-1).
- **IT-005**: stub reporting a below-minimum version → `Skew{newer:false}` state carrying both versions (US-005.AC-1).
- **IT-006**: stub reporting a higher `StatusSchemaVersion` → `Skew{newer:true}` (US-005.EC-1).
- **IT-024**: shell `start_owned` racing a direct fake-daemon start (flock contention) → exactly one process; shell attaches; no kill signals anywhere (US-004.EC-3).
- **IT-026**: `daemon.json` present with dead PID → ladder treats absent → proceeds to start path (US-003.EC-1).
- **IT-027**: two temp homes, daemons on both, `COMPOZY_HOME` pointing at home A → shell resolves A only; never reads or renders B (US-003.AC-3, US-020.EC-1).
- **IT-028**: stub returning 500s → `RuntimeUnhealthy` state; stub fixed → in-state retry succeeds without shell restart (US-003.EC-3, US-006.EC-2).

### Crash and quit contracts

- **IT-007**: attached fake daemon killed → `Disconnected` within the detection interval; daemon restarted externally → reconnect returns to `Product` without shell restart (US-007.AC-1/AC-2/EC-1/EC-3).
- **IT-008**: fake daemon crash-looping on start → exactly 3 supervised attempts with backoff, then terminal error state with diagnostics (US-004.EC-1, US-007.EC-2).
- **IT-009**: shell exits while owning a running fake daemon → daemon still alive and serving after shell PID is gone (US-008.AC-1/AC-2).
- **IT-010**: shell force-killed → relaunch resolves normally (stale `app.json` liveness recovers); attached daemon undisturbed (US-008.EC-2, US-009.EC-2).

### Provisioning recovery

- **IT-011**: kill shell mid-download → relaunch detects incomplete stage → continue → success; postcondition identical to IT-003 (US-002.AC-1/AC-2).
- **IT-012**: artifact server offline for first attempt → `ProvisionNetwork` retry state → server up → retry succeeds without reinstalling the app (US-001.EC-1, US-002.EC-2).

### Updater loops (fixture feed, staging keys)

- **IT-013**: feed with newer version + validly signed artifact → check → download → verify → `ready` (US-014.AC-1).
- **IT-014**: artifact tampered post-signing → verification rejects; nothing staged for install; state reports the failure (US-014.AC-3).
- **IT-015**: malformed feed (missing platform entry / invalid JSON) → diagnosable error state, `last_checked_at` + error visible; no retry storm (US-015.AC-2).
- **IT-025**: `update_check = false` in the home's `config.toml` → no feed request observed across the check interval (fixture server logs zero hits) (ADR-010).

### Runtime update apply (owned)

- **IT-016**: provisioned fake runtime + **signed** `runtime.json` advertising newer version → consent → quiesce sequence observed against the stub drain surface (drain → safe-to-stop → revalidate) → stop (SIGTERM) inside the `update.lock` transaction → swap → start → bound probe confirms new version; marker + journal updated (US-016.AC-1/AC-3, B-004).
- **IT-017**: corrupt archive → verification fails **before** any quiesce or stop: the daemon is never drained, never stopped, old binary untouched and still serving; error state reported. This is the single staging guarantee (N-003) — there is no restore-after-stop path for verification failures.
- **IT-018**: stub `/api/settings/update` with `managed:true` → surface shows recommendation; after stub flips to updated version, state clears to idle with no residual pending (US-017.AC-1/AC-2).
- **IT-029**: migration-interlock journey — fake daemon "advances" a migration marker post-swap then fails to boot → journal classifies post-migration failure → `recovery_required` surfaced in `app.json` and `compozy app status`; old binary NOT auto-restarted; a subsequent newer signed build resolves the state (B-005).
- **IT-030**: quiesce race end-to-end — stub flips safe-to-stop to false between readout and stop → abort observed, undrain called, daemon still serving (B-004).

### Links and single instance

- **IT-019**: validated link event injected while `Product` → webview navigates to `<origin><path>` (US-010.AC-1).
- **IT-020**: second shell process launched with a link argument → argument forwarded to first instance, second exits 0, first navigates; during first-run provisioning the second launch focuses, never starts a parallel provisioning (US-009.AC-1/EC-1, US-002.EC-1). *(Runner: Linux/Windows CI; macOS covered by E2E-007/E2E-023.)*

### Cross-language contracts (Go)

- **IT-021**: Rust integration fixture writes `app.json` through real transitions (including `attaching`, `updating`, `error` with typed error object); Go `compozy app status` parses it → golden report per state; **both sides validate every fixture against `desktop/schema/app-state.schema.json`** — schema drift fails the build (B-007).
- **IT-022**: shared config validation corpus — Go validation results and Rust parse/fail-closed results agree case-by-case (defaults, bounds, malformed, unknown key) (B-011).
- **IT-023**: real filesystem home with provisioned binary + marker → `internal/update` detection through the settings handler → `GET /api/settings/update` responds `install_method:"desktop-app", managed:true` + app-naming recommendation (wired through the existing core-handler test harness).
- **IT-031**: control-socket round trip — Go `compozy app update --check`/`--apply`/`retry`/`diagnose` against the Rust shell's real `app.sock` (integration fixture shell) → verbs execute the same primitives the UI path executes; deterministic errors when the socket is absent/unresponsive (B-006).
- **IT-032**: quiesce-readout contract parity — the status/drain payload's safe-to-stop readout is asserted identically over HTTP and UDS in the core handler suite (no partial surface).

## End-to-End Tests

Journeys run a real shell build against the real daemon binary under isolated `COMPOZY_HOME`s. Platform notes per case; macOS journeys without WebDriver run as scripted-manual platform smoke with recorded evidence per release.

### Newcomer install & first run (US-001, US-002)

- **E2E-001**: fresh OS user, no runtime → install app → open → guided provisioning with visible stages → product UI renders → quit → relaunch lands directly in product (no provisioning) — verify exactly one daemon process and `compozy status` healthy.
- **E2E-002**: network disabled → open app → honest network-required state with retry → enable network → retry from the same screen → provisioning completes (US-001.EC-1).
- **E2E-020**: install build N over existing build N-1 → single app entry, single uninstall record, state preserved (US-001.EC-4). *(Platform smoke: per-OS installer semantics.)*

### CLI power user attach (US-003, US-004, US-008, US-020)

- **E2E-003**: daemon running with an active session + browser tab open → open app → identical workspace/session state as the tab, including local UI state (same origin); no second daemon in the process table (US-003.AC-1, US-020.AC-1).
- **E2E-004**: installed-but-stopped runtime → open app → starting progress → product UI → quit app → `compozy status` still healthy and the session survives (US-004.AC-1, US-008.AC-1/AC-2).
- **E2E-018**: app and browser tab open side by side → create a session in the app → tab reflects it live; act in the tab → app reflects it (US-020.AC-2).

### Connection states (US-005, US-006, US-007, US-013)

- **E2E-005**: pin an old runtime binary in the home → open app → guided incompatibility state naming both versions + recommended action; no product UI rendered (US-005.AC-1).
- **E2E-006**: `kill -9` the daemon while the app shows product UI → disconnected state within the interval → use restart-runtime affordance → product returns with preserved runtime state (US-007).
- **E2E-011**: normal launch shows branded loading with no white flash (frame capture); daemon wedged (SIGSTOP) → bounded loading hands off to the honest error state with diagnostics access (US-013.AC-1/AC-2, US-006.AC-2, US-004.EC-2).

### Native integration (US-009, US-010, US-011, US-012)

- **E2E-007**: app running → launch again from dock/launcher/CLI → existing window focused/unminimized, process count unchanged (US-009.AC-1/AC-2).
- **E2E-008**: with app running, activate `compozyos://open/<existing-session-path>` → window focuses and shows that view; activate a link to a deleted entity → product's own not-found view; activate a malformed link → default view, no dialog (US-010.AC-1/EC-1/EC-2).
- **E2E-023**: with app closed, activate a CompozyOS link → app cold-starts, provision/start states run if needed, and the linked view renders after ready (US-010.AC-2, US-009.EC-1).
- **E2E-009**: click an external `https://` link inside the product UI → OS default browser opens it; app window still on the product; `target=_blank` external behaves identically (US-011).
- **E2E-010**: move/resize window → quit → relaunch restores geometry; then relaunch with saved geometry pointing at a disconnected display profile → centered recovery (US-012).

### Update system (US-014, US-015, US-016, US-017, US-018)

- **E2E-012**: install build N → publish N+1 to the fixture feed → app surfaces "ready" after background download → consent restart → N+1 running and version indicator updated (US-014.AC-1/AC-2). *(Scripted on Linux/Windows; macOS in the release rehearsal.)*
- **E2E-022**: with an update downloaded but unapplied, quit → next launch runs/applies the new version per platform convention; update not lost (US-014.EC-3).
- **E2E-013**: force the apply step to fail (locked install dir fixture) → app reports the failed update and the manual-download path opens the release page; crash-on-new-version scenario leaves diagnostics and the OS-level app still openable (US-015.AC-1/EC-1).
- **E2E-014**: app-provisioned home with agent work in flight → runtime update ready → timing consent prompt → "later" keeps working; "now" stops, applies, restarts, reconnects, and both new versions show in one update surface (US-016.AC-1/AC-2/AC-3).
- **E2E-015**: homebrew-installed runtime → update surface shows availability + exact `brew` recommendation; no binary mtime change anywhere; after the user updates via brew, surface clears (US-017.AC-1/AC-2).
- **E2E-016**: about/update surface shows `beta` + version; no stable channel selector exists anywhere in the UI (US-018.AC-1/AC-2).

### Agent manageability (US-019)

- **E2E-017**: drive the full lifecycle from a terminal: `compozy app status` before install (`installed:false`) → install → `compozy app open` (launches) → status during provisioning (transitional state) → status attached (`running:true`, runtime fields) → `compozy app open /settings` focuses + navigates → kill app → status `running:false`. All with `-o json` asserted against the canonical schema (US-019 all ACs/ECs).
- **E2E-024**: agent-driven update journey — with an update available on the fixture feed: `compozy app update --check` reports it → `compozy app update --apply app` walks consent semantics deterministically → restart → new version in `compozy app status`; then `--apply runtime` on an app-provisioned home walks quiesce + apply + reconnect headlessly (B-006).
- **E2E-025**: recovery-state journey — induced post-migration boot failure (fixture build) → `compozy app status` reports `runtime_state: recovery_required` with typed error → `compozy app diagnose` returns log paths → applying the fixed newer build clears the state (B-005).

### Release integrity (US-021)

- **E2E-019**: release-workflow rehearsal — dry-run without signing secrets hard-fails before any publish step; draft release with full secrets publishes feed + artifacts, then the post-publish verifier downloads `latest.json` + one artifact per platform and validates signatures against the public key; simulate one platform build failure → no feed published, operator gets the explicit evidence (US-021.AC-1/AC-2/EC-1).

### Platform validation gate (PRD constraint)

- **E2E-021**: streaming-load validation on each OS webview (WKWebView, WebView2, WebKitGTK): open the most stream-heavy product screens, measure concurrent SSE/WS connections and UI liveness for 10 minutes → no starvation, no dead streams; record the measured per-origin connection profile in the release evidence. **Release gate — a failure here blocks ship and escalates per TechSpec Known Risks.**
