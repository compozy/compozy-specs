# TechSpec: CompozyOS Desktop App

Companion artifacts: [`_prd.md`](_prd.md) · [`_user_stories.md`](_user_stories.md) · [`_tests.md`](_tests.md) · ADRs 001–012 under [`adrs/`](adrs/). Research corpus: [`analysis/`](analysis/) — implementers read [`06_analysis_tauri-core-verified.md`](analysis/06_analysis_tauri-core-verified.md) and [`07_analysis_tauri-dist-release.md`](analysis/07_analysis_tauri-dist-release.md) (the verified 2026 mechanics/release corpus; they supersede slices 01/03/05 wherever they disagree), plus [`04_analysis_compozy-runtime.md`](analysis/04_analysis_compozy-runtime.md) with the HEAD corrections noted in this spec, and [`_brief.md`](_brief.md). Working Tauri v2 reference implementations (gitignored, local-only — cite by path, never assume present in CI): `.resources/foundry/packages/desktop/` (plugins: dialog/opener/process/shell), `.resources/openfang/crates/openfang-desktop/` (capabilities layout), `.resources/librefang/crates/librefang-desktop/`.

## Executive Summary

The CompozyOS Desktop App is a **pure-Rust Tauri v2 thin shell** in a new top-level `desktop/` directory. One native window loads the web UI **exactly as the daemon serves it** — at the discovery-derived origin when attaching to an existing daemon, `http://localhost:2123` being the default origin for daemons the app starts or provisions (ADR-007): the shell requires zero `web/` changes, API/SSE/WS stay same-origin, and browser users keep their local state. The page receives **no Tauri IPC**; every native affordance — single instance, deep links, window state, honest boot/error screens, auto-update — lives on the Rust side. The shell resolves its runtime **attach-first, provision-if-absent** (ADR-002) using the daemon's existing discovery contract (`daemon.json`, PID+start-time liveness), and provisions by **downloading the runtime at first run** to an app-owned path under `$COMPOZY_HOME` with a durable provenance marker (ADR-008). App updates ride `tauri-plugin-updater` against a channel-scoped feed on `releases.compozy.com` (ADR-004/005); app-provisioned runtime updates reuse the same download pipeline, app-driven (ADR-011). Agent manageability lands as `compozy app open|status` with structured output (ADR-006), and configuration is a minimal `[app]` section in `config.toml` (ADR-010).

Primary trade-offs: first-run provisioning requires network (offline is a designed refusal, not a success path); the in-window page cannot call native APIs in v1 (escalation path documented in ADR-007); and the shell duplicates a bounded slice of download+verify logic in Rust because Windows cannot self-swap a running binary (ADR-011).

Peer review round 1 (BLOCKED, 12 blockers — findings at `qa/peer-review-findings-round1.md`) was fully incorporated: the runtime publication is now signed (ADR-011.5), daemon identity is bound to the discovery record rather than payload shape, runtime mutation runs inside a home-scoped cross-process transaction with durable ownership (ADR-011.6), stop flows through the daemon's quiesce/drain contract (ADR-011.7), updates are migration-aware with an explicit recovery state (ADR-011.8), the agent surface gains update/retry/diagnose control (`compozy app` + local control socket), `app.json`/status share one canonical cross-language schema, public error fields are typed and redaction-bounded, and this workstream absorbs the Compozy→CompozyOS **product-language** hard cut with command identifiers staying `compozy` (ADR-012).

## MVP Boundary

**MVP = the full PRD scope plus the absorbed product-language brand cut, shipped as one workstream** — all nine PRD core features: shell window over daemon UI, attach-first resolution with guided provisioning, complete app auto-update, runtime update surfacing per ownership, native integration (single instance, deep links, window state, external links, flash-free launch), `compozy app` CLI surface (open/status/update/retry/diagnose), beta channel with reserved stable identity, release-pipeline integration with fail-closed gates, the `[app]` config lifecycle — plus the Compozy→CompozyOS product-language rename sweep (ADR-012).

**Post-MVP (explicitly deferred, out of this spec):** bundling `frontendDist` with page IPC (escalation path, ADR-007); tray/menubar background mode; multiple windows; stable channel activation (identity reserved only, ADR-005); Mac App Store or any store distribution; Windows arm64; renaming command identifiers (`compozy` binary, `COMPOZY_*` env, module path, packages — explicitly kept, ADR-012); managing remote daemons from the shell.

**Out of scope forever for this surface:** rebuilding any product UI natively; a Rust HTTP proxy in front of the daemon; Compozy-hosted update relays.

## System Architecture

### Component Overview

```
┌────────────────────────────── desktop/ (new, Rust) ──────────────────────────────┐
│ Tauri v2 shell "CompozyOS"                                                       │
│                                                                                  │
│  boot/state machine ──► runtime resolver ──► window/navigation controller        │
│        │                    │  ├─ discovery (daemon.json + liveness)             │
│        │                    │  ├─ identity probe + version handshake (/api/status)│
│        │                    │  ├─ supervisor (start owned daemon, readiness)     │
│        │                    │  └─ provisioner (download→verify→install→marker)   │
│        │                                                                         │
│  app updater (tauri-plugin-updater ⇄ releases.compozy.com/desktop/<ch>/latest.json)│
│  runtime updater (consumes GET /api/settings/update; app-driven apply)           │
│  deep-link handler (compozyos:// validate → queue → navigate)                    │
│  app record writer ($COMPOZY_HOME/app.json, atomic)                              │
│  config reader ($COMPOZY_HOME/config.toml [app])                                 │
│  bundled shell pages (loading / provisioning / error / skew / update)            │
└──────────────────────────────────────────────────────────────────────────────────┘
        │ HTTP (localhost only)                        │ launches / reads
        ▼                                              ▼
   Go daemon (unchanged surface)                 compozy CLI (new `app` verb)
   /api/status · /api/settings/update            reads app.json + liveness
   daemon.json · SPA at localhost:2123           opens via scheme URL
```

Data flows: the shell **reads** daemon truth (discovery record, status, update state) and **never writes daemon-owned state**. The daemon knows nothing about the shell except the provenance marker its install-method detection reads (ADR-011). The web UI inside the window talks to the daemon exactly as a browser tab does.

### Architectural Boundaries

New surface layout (one responsibility per file; hard cap 500 lines each, split decided now):

```
desktop/
  src-tauri/
    Cargo.toml                    # crate compozyos-desktop
    tauri.conf.json               # base config; SINGLE durable identity: identifier com.compozy.os, name CompozyOS
    tauri.channel.conf.json       # CI-generated overlay: injects ONLY the channel feed endpoint (see channel model)
    capabilities/boot.json        # minimal grants for the bundled boot window ONLY; main window gets NO capability, no remote grants
    src/
      main.rs                     # entry; plugin registration (single-instance FIRST)
      state.rs                    # ShellState machine + transitions (no IO)
      home.rs                     # COMPOZY_HOME resolution (mirrors internal/config/home.go rules)
      config.rs                   # [app] section reader + defaults
      record.rs                   # app.json atomic writer + liveness fields
      runtime/discovery.rs        # daemon.json read + PID/start-time liveness
      runtime/probe.rs            # bound identity probe (status ⇄ daemon.json) + version handshake
      runtime/resolver.rs         # install resolvers: app-owned → daemon.json-live → PATH; per-platform app registration
      runtime/supervisor.rs       # start/stop owned daemon, readiness poll, crash backoff
      runtime/mutation.rs         # home-scoped update-lock transaction + commit journal + crash recovery
      runtime/provision.rs        # download→verify→stage→rename pipeline + provenance marker
      runtime/artifacts.rs        # signed runtime.json client: minisign verify → schema-heads → sha256
      runtime/quiesce.rs          # daemon drain contract client: drain → safe-to-stop → revalidate → undrain
      update/app_update.rs        # updater plugin orchestration + restart watchdog
      update/runtime_update.rs    # consent flow composing quiesce + mutation + pipeline for owned runtimes
      control.rs                  # local control socket ($COMPOZY_HOME/app.sock): CLI↔shell verbs
      errors.rs                   # typed ShellErrorCode + bounded safe_message + redaction layer (single sink gate)
      links.rs                    # scheme payload validation + last-wins queue
      nav.rs                      # navigation fencing on the main window; external→browser
      windowing.rs                # two-window orchestration (boot ⇄ main), window-state hooks, flash-free show, bounded load deadline
      logging.rs                  # tauri-plugin-log setup; platform log paths; redaction-gated
    pages/                        # bundled shell pages (static HTML/CSS, DESIGN.md token values)
  schema/
    app-state.schema.json         # canonical versioned schema for app.json + AppStatusReport
    fixtures/                     # shared cross-language fixture corpus (Rust and Go both validate)
  tests/                          # Rust integration tests (stub daemon HTTP server)
```

Import rules inside the crate: `state.rs` is pure (no IO) and imported by everything; `errors.rs` is the single redaction gate every sink (record, logging, control) imports; `runtime/*` never imports `update/*`, `control.rs`, or `nav.rs`; `update/runtime_update.rs` composes `runtime/quiesce.rs` + `runtime/mutation.rs` + `runtime/artifacts.rs` + `runtime/supervisor.rs` (it owns the sequence, not the primitives); `main.rs` is the only composition point (mirror of the daemon's `daemon/` discipline).

Go packages touched:

- `internal/cli/app.go` + `internal/cli/app_types.go` + `internal/cli/app_control.go` — `compozy app open|status|update|retry|diagnose` verb tree (registered in `internal/cli/root.go` beside `newOpenCommand`); the control-socket client lives in its own file.
- `internal/config/` — new `AppConfig` struct + `[app]` defaults/validation/docs (per-section file pattern; fail-closed invalid-config policy per ADR-010 amendment).
- `internal/update/types.go` + `internal/update/detect.go` + the recommendation/state mapper in `internal/update/manager_state.go` — `InstallMethodDesktopApp`, provenance-marker branch, and the `desktop-app` recommendation string, covered in the existing detect/recommendation matrix suites (N-005).
- `internal/api/contract` + `internal/api/core` (+ HTTP/UDS registration) — **one bounded contract change:** the status/drain surface exposes the daemon's quiesce readout (drain state + safe-to-stop) if not already public; closes core → HTTP → UDS → codegen → generated TS in the same change (no partial surface).
- **Brand-cut sweep (ADR-012):** product-language occurrences in `web/` display strings, `packages/site`, `COPY.md`, `docs/_memory/glossary.md`, `PRODUCT.md`, `skills/compozy/` — prose/strings only; no behavior change.
- **Not touched:** storage/migrations (no schema change), extension/hook/tool registries, network protocol.

Build/CI boundaries: `desktop/` sits **outside** the Bun/turbo graph and outside `mage Boundaries` (Rust is not in the Go import graph). New make targets `desktop-dev`, `desktop-build`, `desktop-test`, `desktop-lint` (cargo fmt+clippy -D warnings); a `desktop` job matrix of **three build jobs** — macOS `universal-apple-darwin` on a pinned `macos-15` runner (the `macos-13` x64 runner class is retired; the universal binary serves both `darwin-aarch64` and `darwin-x86_64` feed entries), Linux x64, Windows x64 — joins the existing `release.yml` under the same tag/channel plan via `tauri-action@v1`. The feed still carries all four platform keys (US-021 validation unchanged).

## Implementation Design

### Core Interfaces

Shell state machine (Rust — the type every module depends on):

The state vocabulary is canonical across `ShellState`, `app.json`, `AppStatusReport`, and the PRD's BR-20 list — one enum, three projections, mechanically validated against `desktop/schema/app-state.schema.json` (B-007):

```rust
pub enum ShellState {
    Resolving,                                  // home + discovery in progress
    Provisioning { stage: ProvisionStage },     // first-run pipeline (US-001/002)
    Starting { attempt: u8 },                   // owned daemon spawn + readiness (US-004)
    Attaching,                                  // bound identity probe + version handshake (US-003/005)
    Product { origin: Url, owned: bool },       // window on daemon origin
    Updating { target: UpdateTarget },          // app or runtime apply in progress (US-014/016)
    Disconnected { cause: DisconnectCause },    // runtime died while open (US-007)
    Skew { runtime: Version, needed: VersionReq, newer: bool },   // US-005
    ShellError { error: ShellError },           // US-006
}

pub enum ProvisionStage { Download { pct: u8 }, Verify, Install, Start, Ready }
pub enum UpdateTarget { App, Runtime }

/// The ONLY error shape that reaches app.json, logs, control socket, or CLI (B-012):
/// typed code + producer-authored bounded message. Raw response bodies, env values,
/// and unfiltered log tails never enter this struct; `errors.rs` is the single gate.
pub struct ShellError {
    pub code: ShellErrorCode,
    pub safe_message: String,   // producer-authored, redaction-checked, ≤ 512 bytes
    pub log_path: PathBuf,      // pointer to diagnostics, never their content
}

pub enum ShellErrorCode {                       // stable identities (agents branch on these)
    PortConflictForeign, RuntimeStartFailed, ProvisionNetwork, ProvisionDiskSpace,
    ProvisionPermission, RuntimeUnhealthy, HandshakeFailed, LoadDeadlineExceeded,
    UpdateLockHeld, QuiesceFailed, MigrationRecoveryRequired, ConfigInvalid,
}
```

Bound identity probe (B-002) — payload shape is never identity; the response must agree with the already-validated discovery record:

```rust
/// Adapter over the real wire contract (internal/api/contract.StatusPayload):
/// `schema_version` is a DATE STRING constant (e.g. "2026-07-16"), daemon fields are
/// nested: daemon.version, daemon.pid, daemon.started_at, daemon.http_host/http_port.
pub struct BoundDaemonIdentity { pub status: StatusPayload, pub record: DaemonRecord }

/// Classification rules, all required for Identity::Compozy:
/// decode succeeds AND status.daemon.pid == record.pid AND start times agree AND
/// http_port == record.port AND (when the shell spawned it) pid == child pid.
/// A well-formed payload failing ANY binding → Identity::Foreign (rejected, US-003.EC-2).
pub enum Identity { Compozy(BoundDaemonIdentity), Foreign, Unreachable }
```

Runtime resolution with explicit install resolvers (B-008):

```rust
pub trait RuntimeResolver {
    /// Attach-first ladder: healthy bound daemon → start installed → provision (ADR-002).
    /// Never returns a half-state: every failure maps to a ShellState.
    fn resolve(&self, home: &CompozyHome) -> Resolution;
}
/// Runtime binary precedence (deterministic, tested per platform):
/// 1. app-owned path (<home>/bin/compozy[.exe]) validated against provenance
/// 2. binary of a live daemon.json process (attach only — never mutated)
/// 3. operator install on PATH (login-shell PATH resolution; GUI apps get no shell env)
/// App installation truth comes from platform registration (macOS bundle query by
/// identifier / Windows uninstall key / Linux .desktop entry), NEVER from app.json.
pub enum Resolution {
    Attached { identity: BoundDaemonIdentity, owned: bool },
    NeedsStart { binary: PathBuf, source: BinarySource },
    NeedsProvision,
    Failed { error: ShellError },
}
pub enum BinarySource { AppOwned, OperatorPath }
```

CLI verb (Go — `internal/cli/app.go`, registered in `registerRootCommands`):

```go
// parent "app": open, status, update (--check|--apply), retry, diagnose (B-006)
func newAppCommand(deps commandDeps) *cobra.Command

// app_types.go — stable structured-output schema (human/json/toon via writeCommandOutput).
// Rust writer and this Go reader are BOTH validated against desktop/schema/app-state.schema.json
// and its shared fixture corpus; drift fails the build (B-007, IT-021).
type AppStatusReport struct {
    SchemaVersion int             `json:"schema_version"` // starts at 1; unknown major → deterministic app_state_schema_unknown error
    Installed     bool            `json:"installed"`      // from platform registration (B-008), never from app.json
    AppVersion    string          `json:"app_version,omitempty"` // from the installed bundle/registration
    Channel       string          `json:"channel,omitempty"`
    Running       bool            `json:"running"`        // app.json liveness (PID + start-time)
    PID           int             `json:"pid,omitempty"`
    State         string          `json:"state,omitempty"` // resolving|provisioning|starting|attaching|product|updating|disconnected|skew|error
    Error         *AppErrorReport `json:"error,omitempty"` // present iff state == "error"
    Runtime       AppRuntimeState `json:"runtime"`
    Update        AppUpdateState  `json:"update"`
}
type AppErrorReport struct {
    Code        string `json:"code"`         // ShellErrorCode verbatim
    SafeMessage string `json:"safe_message"` // bounded, redaction-gated (B-012)
    LogPath     string `json:"log_path"`
}
type AppRuntimeState struct {
    Attached bool   `json:"attached"`
    Owned    bool   `json:"owned"`
    Version  string `json:"version,omitempty"`
}
type AppUpdateState struct {
    AppState         string `json:"app_state"`          // idle|checking|downloading|ready|failed
    AppAvailable     string `json:"app_available,omitempty"`
    RuntimeState     string `json:"runtime_state"`      // idle|available|applying|recovery_required|failed|managed
    RuntimeAvailable string `json:"runtime_available,omitempty"`
    LastCheckedAt    string `json:"last_checked_at,omitempty"`
    LastError        *AppErrorReport `json:"last_error,omitempty"`
}
```

Control channel (B-006): the running shell listens on `$COMPOZY_HOME/app.sock` (0600, versioned JSON-RPC-style protocol) exposing exactly the primitives the shell UI itself calls: `navigate(path)`, `update.check`, `update.apply(target)`, `retry`, `diagnose` (returns log paths + last errors). `compozy app update|retry|diagnose` are thin clients of this socket; `compozy app open` keeps the scheme handoff for cold start and uses the socket when the shell is already running. UI and CLI converge on one primitive per action — no CLI-only or UI-only behavior.

Deterministic CLI errors (stable identities, `-o json` carries `{"error": {"code": ...}}`): `app_not_installed`, `app_not_running` (for verbs requiring the control socket), `app_launch_failed`, `invalid_target_path`, `app_state_schema_unknown`, `app_control_unavailable` (socket present but unresponsive). `compozy app status` with the app not running is **not** an error — it reports `running: false`, with `installed` derived from platform registration and update state from the last-written `app.json` (truthful transitional states, US-019.EC-2).

Config (Go — `internal/config`):

```go
type AppConfig struct {
    UpdateCheck         bool          `toml:"update_check"`          // default true
    UpdateCheckInterval time.Duration `toml:"update_check_interval"` // default 6h; bounds [15m, 168h]
}
// Config gains: App AppConfig `toml:"app"`; defaults in DefaultWithHome; validation rejects
// out-of-bounds intervals with the exact key path in the error.
```

Install-method detection (Go — `internal/update`):

```go
const InstallMethodDesktopApp InstallMethod = "desktop-app" // types.go, beside existing methods

// detect.go: before generic heuristics, when the executable path equals
// <home>/bin/compozy[.exe] and <home>/bin/.desktop-provenance.json parses and matches
// the binary hash → InstallMethodDesktopApp, managed=true,
// recommendation "Update via the CompozyOS desktop app."
// Mismatch or unreadable marker → fall through to existing heuristics (inconclusive ⇒ PRD BR-8).
```

### Data Models

No SQLite entity is added anywhere in this feature. **Side-table-vs-JSON decision:** not applicable in its usual daemon sense — the shell introduces no daemon-matchable state and writes nothing into any `metadata_json`; its records are single-writer local files (JSON) owned by the shell process, mirroring the established `daemon.json` discovery pattern. Each file and field:

**`$COMPOZY_HOME/app.json`** — shell state record (atomic write-then-rename; consumed by `compozy app status`). Its shape is governed by the canonical schema `desktop/schema/app-state.schema.json` — the Rust writer and Go reader both validate against it plus the shared fixture corpus, and drift fails the build (B-007):

- **schema_version** — int, starts at 1; consumers reject unknown majors deterministically (`app_state_schema_unknown`).
- **pid**, **started_at** — process identity for liveness (same PID+start-time algorithm as `daemon.json`; a dead PID means "not running" regardless of file presence). Liveness and last-known state are the ONLY authorities this file carries — installation identity comes from platform registration (B-008).
- **app_version**, **channel** — as last written; informational (authoritative app version comes from the installed bundle).
- **state** — `resolving|provisioning|starting|attaching|product|updating|disconnected|skew|error` (canonical vocabulary, BR-20); written on every transition so mid-provisioning/mid-update queries are truthful (US-019.EC-2).
- **error** — `{code, safe_message, log_path}` present iff state is `error`; typed and redaction-gated (B-012); never free-form.
- **runtime** — `{attached, owned, version, origin}`; `owned` is a status projection — ownership **authority** is durable: it is reacquired at any time from the provenance marker + executable path/hash agreement, never from this file or from process memory (B-003).
- **update** — `{app_state, app_available, runtime_state, runtime_available, last_checked_at, last_error}` for US-015.AC-2 honesty; `runtime_state` includes `recovery_required` (B-005).

**`$COMPOZY_HOME/update.lock`** — home-scoped runtime-mutation lock (B-003): flock + `{pid, started_at}` payload with stale detection (same algorithm as `daemon.lock`). Held across the entire quiesce→stop→swap→start interval by whichever process mutates the runtime; `compozy daemon start` acquisition and shell acquisition contend on the same file, and the holder re-probes runtime state after acquiring before acting.

**`$COMPOZY_HOME/bin/.update-journal.json`** — update-phase commit journal (B-003/B-005): `{phase: staged|stopped|swapped|migrating|started|done, from_version, to_version, staged_path, written_at}`, fsync'd before each phase transition. Crash recovery reads the journal: pre-`swapped` phases roll back cleanly (old binary intact); post-start failure with migrations possibly advanced surfaces the explicit `recovery_required` state — the old binary is never silently restarted (ADR-011.8).

**`$COMPOZY_HOME/bin/.desktop-provenance.json`** — ownership marker (ADR-008/011; read by `internal/update/detect.go`):

- **installed_by** — literal `"desktop-app"`; the discriminator.
- **app_version**, **runtime_version**, **channel**, **installed_at** — audit fields for support and status.
- **binary_sha256** — hash of the installed runtime binary; detection treats mismatch as inconclusive → not app-owned (BR-8).

**Window-state store** — owned by `tauri-plugin-window-state` in the platform app-data dir; not ours to schema. Invalid-geometry recovery (off-screen, sub-minimum, removed monitor) validates restored bounds against current work areas before applying and falls back to centered defaults (US-012).

**`releases.compozy.com/desktop/<channel>/latest.json`** — Tauri updater feed (plugin-defined schema): `version`, `notes`, `pub_date`, `platforms.{darwin-aarch64,darwin-x86_64,linux-x86_64,windows-x86_64}.{signature,url}` — URLs point at own-domain artifacts (ADR-004). Generated and schema-validated in CI before publish; a missing platform entry fails the release (US-021.EC-1).

**`releases.compozy.com/desktop/<channel>/runtime.json` + `runtime.json.minisig`** — **signed** runtime pointer manifest (ours; B-001/ADR-011.5). The pipeline verifies the minisign signature (same update-feed keypair, pubkey embedded in the shell) over canonical manifest bytes **before** trusting any field; only then is the archive digest meaningful:

- **schema_version** — int, starts at 1.
- **version** — the channel's current runtime version (from the existing release-plan channel outputs — one channel truth, ADR-005). Must exceed the installed version or the manifest is a no-op (BR-10).
- **platforms.<target>.{url, sha256, size}** — per-platform runtime archive on own-domain storage; the `sha256` is bound inside the signed manifest, so a swapped URL+digest pair fails signature verification.
- **schema_heads** — the runtime version's migration-stream heads (per physical store), enabling the migration-aware preflight and the recovery-state decision (B-005). Generated by the release pipeline from the runtime build.

**`config.toml` `[app]` keys** — see Core Interfaces (`AppConfig`); rationale per key: `update_check` exists for isolated-QA and air-gapped machines (structured off switch, ADR-010); `update_check_interval` exists because QA needs short cycles and operators need calm defaults — bounds guard against accidental hammering (15m floor) and dead checks (7d ceiling).

### API Endpoints

Consumed existing surfaces:

| Surface | Use |
| --- | --- |
| `GET /api/status` | **Bound** identity probe + version handshake — decoded via the real contract shape (date-string `schema_version`, nested `daemon.*`) and bound to `daemon.json` (B-002) |
| `GET /api/settings/update` | Runtime update truth: `{available, latest_version, install_method, managed, recommendation}` |
| Daemon drain surface (`/api/drain`, `/api/undrain` + drain state) | The quiesce contract (ADR-011.7): close admission → settle claims/session execution → safe-to-stop readout → revalidate before SIGTERM → undrain on abort |
| `$COMPOZY_HOME/daemon.json` | Discovery: `{pid, port, started_at}` + PID/start-time liveness |
| `compozy daemon start` semantics | The supervisor starts owned daemons with the same detached+readiness contract the CLI uses |

**One bounded daemon contract change:** the quiesce readout (drain state + safe-to-stop, including "authoritative claims settled") must be machine-readable by the shell. If the existing drain surface does not already expose it, the status/drain payload gains it — closed through `internal/api/contract` → `internal/api/core` → HTTP → UDS → codegen (OpenAPI + generated TS) → docs in the same change; no partial surface. The shell never inspects `task_runs` (B-004).

New non-HTTP public surfaces:

| Surface | Shape |
| --- | --- |
| `compozy app open [path]` | Launches or focuses the app (scheme handoff cold, control socket warm); optional validated product path (ADR-009). Exit 0 on handoff; `app_not_installed` otherwise. |
| `compozy app status` | `AppStatusReport` via `-o human\|json\|toon`; schema-validated |
| `compozy app update [--check\|--apply <app\|runtime>]` | Drives the shell's update primitives over the control socket; deterministic errors (`app_not_running`, `app_control_unavailable`) |
| `compozy app retry` / `compozy app diagnose` | Retry the current designed-error state / return log paths + last typed errors (B-006) |
| `$COMPOZY_HOME/app.sock` control protocol | Versioned local socket (0600): `navigate`, `update.check`, `update.apply`, `retry`, `diagnose` — the same primitives the shell UI calls |
| `compozyos://` scheme (prod) / `compozyos-dev://` (dev) | Payload = product path + query/fragment; validation per ADR-009 (absolute path, no scheme/authority, no traversal) |
| `[app]` config section | Keys above; full lifecycle (defaults, validation, `compozy config`, generated docs); invalid config fails the update consumer closed (ADR-010 amended) |
| `latest.json` / `runtime.json`(+`.minisig`) feeds | Published per channel on `releases.compozy.com/desktop/<channel>/`; `stable/` path reserved and unpublished (ADR-005) |

## Integration Points

- **Daemon (localhost HTTP)** — read-only consumption of status/update surfaces; the identity probe requires a well-formed `StatusResponse` before the window ever navigates to the origin (US-003.EC-2 squatter defense). Timeouts on every call (no default clients without deadlines — same discipline as the Go rule). Verified post-gateway facts (runtime verification vs HEAD `b1885a15`): the local listener never applies device auth, stream tickets, or the pairing gate — localhost is the root of trust, and the window at the daemon origin inherits full trusted-surface authority (including gateway management routes) with zero credentials; the web app's `GatewayAccessBoundary` is inert on localhost by construction. Conversely, the daemon's engine-level CORS hard-403s any non-`http(s)` origin (`internal/api/httpapi/middleware.go`), which is why bundling into the Tauri asset protocol is unworkable (ADR-007).
- **Release distribution (`releases.compozy.com`)** — app feed, app artifacts, runtime manifest, runtime archives, all under `desktop/<channel>/`; uploaded by the release pipeline with the org's existing object-storage credentials (ADR-004). Retry strategy: updater checks are periodic and silently retried (US-014.EC-1); provisioning downloads offer explicit retry (US-002).
- **GitHub Releases** — canonical archive only; never in the hot update path (ADR-004).
- **Tauri plugins** (baseline versions as of 2026-08-09: `tauri` 2.11.5, `tauri-plugin-updater` 2.10.1; re-pin at implementation start): `single-instance` (registered first; second-launch args forwarded), `deep-link`, `window-state`, `updater`, `opener`, `log`.
- **OS trust chains** — macOS Developer ID + notarization + stapling (note: Tauri does not notarize/staple the DMG itself — tauri#7533 open — so the pipeline runs `notarytool`+`stapler` on the DMG as an explicit step; the update payload is the signed `.app.tar.gz`, validated by the updater signature on the update path). Windows: the org's provisioned Azure signing service (the product was renamed **Azure Artifact Signing**, CLI `artifact-signing-cli` — the legacy repo's signer script predates the rename and is stale reference only). Linux: `tauri-plugin-updater` can update deb/rpm too but only via `pkexec`/`sudo` privilege escalation — v1 policy is **AppImage-only auto-update**; `.deb` ships install-only and reports as managed (recommendation, no self-update). All signing reuses org credentials with variable remapping (ADR-004).

## Safety Invariants

Ownership and lifecycle invariants, numbered — every one is testable and none has an override:

1. The shell never stops, restarts, replaces, or modifies a runtime it did not start or provision — no exception, including during updates (BR-1).
2. Quitting the shell never stops any runtime, owned or not; stopping is exclusively the runtime's own control surface (BR-2).
3. At most one shell instance per user session; a second launch forwards its arguments to the first and exits (BR-3).
4. `COMPOZY_HOME` resolution follows the runtime's rules exactly; no hardcoded home anywhere in the shell (BR-4).
5. A discovery record whose process fails PID+start-time liveness is "no runtime" — never attached, never signaled (BR-5); and a status response is trusted only when **bound** to that record (PID, start time, port agree) — a well-formed payload without binding is foreign and rejected (B-002).
6. An update artifact is applied only after verification succeeds — minisign signature for the app; for the runtime, minisign signature over `runtime.json` first, then the archive `sha256` bound inside the signed manifest; unverifiable ⇒ never applied, always reported (BR-6, B-001).
7. App updates apply only on user-consented restart; the shell never force-restarts itself (BR-7).
8. Runtime binaries are written only under `$COMPOZY_HOME/bin/` inside the home-scoped update-lock transaction, via journaled stage-then-rename; a failed download, verify, or install leaves the previous binary and marker untouched, and a crash at any phase is recoverable from the journal (atomic-or-absent, ADR-011.6).
9. A runtime update never applies while agent work is in flight without explicit timing consent (BR-9), and stop happens only through the daemon's quiesce contract with a revalidation immediately before SIGTERM — the shell never infers queue safety or touches `task_runs` (B-004).
10. Downgrades are never offered — app (updater semantics) or runtime (manifest version must exceed installed) (BR-10).
11. No Tauri capability grants IPC to the daemon origin or any remote origin; capability files cover bundled shell pages only (ADR-007).
12. In-window navigation is fenced to the daemon origin + bundled pages; every other destination — including `window.open` and `target=_blank` — opens in the OS default browser (BR-15).
13. Deep-link payloads that fail shape validation are discarded to the default view; a validated payload can only ever produce a product-path navigation (BR-16).
14. Every waiting state is bounded: readiness polls, first-paint, provisioning stages, and update watchdogs all have deadlines that land in a designed state — no permanent spinner, no restart loop (BR-13/BR-14: bounded attempts with terminal failed state).
15. The shell writes no daemon-owned state; `app.json` and the provenance marker are shell-owned single-writer files, and daemon truth always wins over shell caches.
16. Every shell state — including transitional and error states — is readable via `compozy app status` with stable vocabulary (BR-20), and every shell control primitive (update check/apply, retry, diagnose) is CLI-operable through the control socket — no UI-only capability (B-006).
17. Exactly one process mutates the runtime installation at a time: the `update.lock` transaction covers quiesce→stop→swap→start, contends with `compozy daemon start`, detects stale holders by PID+start-time, and re-probes after acquisition (B-003).
18. App-provisioned ownership is durable and evidence-based: reacquired from the provenance marker + executable path/hash agreement, never from process memory or `app.json`; disagreement ⇒ not owned ⇒ recommendation-only (B-003, BR-8).
19. Public error surfaces are safe by construction: only typed `ShellErrorCode` + bounded producer-authored `safe_message` cross into `app.json`, logs, the control socket, or CLI output — raw response bodies, env values, tokens, and unfiltered log tails are structurally excluded and adversarially tested (B-012).
20. After a runtime update in which migrations may have advanced, the old binary is never silently restarted; the explicit `recovery_required` state (journal-derived) is surfaced to humans and agents until resolved by a later signed build or explicit operator action (B-005).

## Impact Analysis

| Component | Impact Type | Description and Risk | Required Action |
| --- | --- | --- | --- |
| `desktop/` | new | Shell crate + bundled pages + canonical schema dir + control socket; risk contained by thin-shell rule and file split | Build per Architectural Boundaries |
| `internal/cli` | modified (additive) | `newAppCommand` (open/status/update/retry/diagnose) + control-socket client + 3 new files; low risk | Implement verbs + tests |
| `internal/config` | modified (additive) | `[app]` section, full lifecycle, fail-closed invalid policy; low risk | Struct, defaults, validation, docs, tests |
| `internal/update` | modified (additive) | New enum value + detect branch + recommendation mapper entry; medium risk (managed-install semantics) | Marker branch + detect/recommendation matrix tests |
| `internal/api/contract` + `internal/api/core` | modified (bounded) | Quiesce readout on the status/drain surface (if absent today); contract change → codegen ripple | Close core→HTTP→UDS→codegen→TS→docs in one change |
| `.github/workflows/release.yml` | modified | Desktop build jobs (macOS universal, Linux x64, Windows x64), signing env remap + DMG stapling, feed + **signed runtime manifest** generation + validation, own-domain upload; medium risk (release path) | Fail-closed gates per US-021 |
| `web/` | modified (brand strings only) | Product-language occurrences → CompozyOS (ADR-012); zero behavior/API change — the shell still requires no web changes | Rename-sweep task; designer lane for brand marks |
| `packages/site` | modified | Install page, desktop getting-started, generated CLI/config docs, `cli/meta.json` nav entry, **site-wide product-language sweep** | Docs + rename tasks |
| `COPY.md` · `docs/_memory/glossary.md` · `PRODUCT.md` | modified | Product = CompozyOS; `compozy` codified as command identifier (ADR-012 boundary) | Same-batch as sweep |
| `skills/compozy/` | modified | Desktop surface + `compozy app` verbs + product-language rename | Update at ship task |
| `docs/qa/scenarios/` | new files | Install/attach/update/deep-link/single-instance/recovery scenarios (content-addressed, mapped to PRD/test IDs) | QA pair tasks |
| Storage/migrations, extension registries, network protocol | none | No schema change (journal/lock are files, not tables); no extension contract change — verified surfaces listed in Extensibility Plan | — |

## Testing Approach

Strategy only — the exhaustive case catalog is [`_tests.md`](_tests.md).

- **Rust unit** (`cargo test`, in `desktop-test` make target and CI lane): pure logic — state-machine transitions, link validation, home resolution, config parsing/defaults, manifest verification, geometry recovery. No network, no processes; IO behind traits with in-memory fakes at the boundary only.
- **Rust integration** (`desktop/tests/`, same lane): resolver ladder and handshake against a **stub daemon HTTP server** (canned `/api/status`, `/api/settings/update`, temp `daemon.json`) plus a **local updater fixture feed** (staging keypair, dummy artifacts) proving check→download→verify and rejection of bad signatures/malformed feeds. Supervisor tests spawn a tiny fake-daemon script honoring SIGTERM.
- **Go unit** (existing suites): `compozy app` verbs (status against fixture `app.json` states, open handoff errors, control-socket client errors), `AppConfig` lifecycle in the config package's canonical validation suite (shared fixture corpus with the Rust side), `InstallMethodDesktopApp` in the existing detect matrix suite **and** the recommendation/state mapper matrix (N-005), quiesce-readout contract cases in the core handler suite.
- **Cross-language contract gates:** `desktop/schema/app-state.schema.json` + `desktop/schema/fixtures/` are validated by both `cargo test` and the Go suites — schema drift or fixture disagreement fails the build (B-007); the config validation corpus works the same way (B-011).
- **End-to-end**: attach/spawn/provision/update journeys run as desktop QA charters under isolated `COMPOZY_HOME`s (real daemon binary, real shell build); `tauri-driver` (WebDriver) exists for Linux/Windows only, so scripted E2E runs on those in CI where feasible and macOS coverage lands in the platform smoke matrix executed per release (real-OS validation is a PRD launch constraint, including SSE-load validation on WKWebView/WebView2).
- **Release pipeline verification**: CI asserts every updater artifact has a `.sig`, validates `latest.json`/`runtime.json` schemas and platform completeness, and runs a signature verify with the public key before publish; a draft-release acceptance run exercises the real feed before first public ship.
- Environment dependencies: staging minisign keypair (never the production key) for fixtures; runners for the 3-job build matrix (macOS universal, Linux x64, Windows x64); isolated homes per the worktree/QA isolation rules.

## Development Sequencing

### Build Order

1. **Shell skeleton + window strategy** — crate scaffold, config/identity per channel overlay, state machine, bundled pages, navigation fencing, flash-free show. No runtime logic yet (window against a hand-started daemon).
2. **Runtime resolution** — home resolution, discovery+liveness, identity probe, version handshake, supervisor (start owned, readiness, crash backoff, quit contract). Depends on 1.
3. **Provisioning pipeline** — `runtime.json` client, download/verify/stage/rename, provenance marker, resumable guided flow. Depends on 2.
4. **Canonical state schema + Go surfaces** — `desktop/schema/app-state.schema.json` + fixture corpus, `app.json` record writer (shell side), `compozy app open|status`, `[app]` config lifecycle (fail-closed policy), `InstallMethodDesktopApp` detection + recommendation mapper. Depends on 2 (record schema), parallel with 3.
5. **Quiesce contract + runtime-mutation transaction** — daemon quiesce readout (contract change if needed, closed end-to-end), `update.lock` transaction, commit journal + crash recovery, durable ownership reacquisition. Depends on 2; blocking for 7.
6. **App auto-update** — updater plugin wiring, consent/restart flow, watchdog, failure fallback UX. Depends on 1; feed fixtures independent of 3/4.
7. **Runtime update surfacing** — consume `/api/settings/update`, ownership split UX, app-driven apply (quiesce→stop→pipeline→start) with timing consent, migration-aware journal + `recovery_required` state. Depends on 3+4+5+6.
8. **Control socket + native integration completion** — `app.sock` protocol + `compozy app update|retry|diagnose`, single instance + arg forwarding, deep links (schemes, cold-start queue), window-state with recovery. Depends on 1 (window), 2 (readiness queueing), 4 (verbs).
9. **Brand hard cut (ADR-012)** — product-language sweep across `web/` strings, `packages/site`, `COPY.md`, glossary, `PRODUCT.md`, `skills/compozy/`, with occurrence check; independent of 1–8, lands before release tasks so shipped surfaces carry one name.
10. **Release pipeline** — `release.yml` desktop jobs, signing/notarization + DMG stapling, feed + signed runtime manifest generation+validation, own-domain upload, fail-closed gates, custody runbook. Depends on artifacts existing (1–8 buildable); admin gates below.
11. **Docs + official skill + QA tail** — site pages, generated CLI/config docs + `cli/meta.json`, `skills/compozy/` update, QA scenarios + charters walk.

### Technical Dependencies

- **Admin gate (critical path, needs owner + date before step 8):** org-secret visibility grant to this repo (macOS + Windows signing) and update-feed keypair generation with the custody/rotation runbook written and reviewed — launch gates per ADR-004/US-021.AC-3.
- Object-storage upload credentials wired into `release.yml` (exist at org level per ADR-004).
- Runner availability for the build matrix (pinned `macos-15` for the universal build, Linux x64, Windows x64).
- Baseline dependency pin (Tauri 2.11.x + plugins) re-verified at implementation start.

## Monitoring and Observability

- **Shell logs** — `tauri-plugin-log` to the platform log dir; the error state's "diagnostics" affordance opens/points at this path plus the runtime's own log location. Log lines carry the state-machine transition (`state=provisioning stage=verify`), update outcomes, and deep-link rejections (payload shape only — never full hostile payloads at info level).
- **`app.json`** — the machine-readable pulse: state, update states, `last_checked_at`, `last_error`. Agents poll `compozy app status`; no push channel in v1.
- **Release observability** — own-domain CDN/object logs are the only update metrics (privacy posture: zero telemetry, no new data collection); the release pipeline's post-publish verification is the health check for the feed itself.
- No daemon events are emitted by the shell (it is not a daemon-side actor); runtime update applies are visible through the daemon's own status/version surfaces.

## Technical Considerations

### Key Decisions

Recorded as ADRs (list below). Spec-level decisions not warranting their own ADR:

- **Windows updater `installMode`: `"passive"`** with a mandatory pre-consent prompt — the NSIS installer force-exits the app, so consent is collected before install ever starts (brief R3); the restart watchdog falls back to a manual-download affordance if the app fails to relaunch (US-015.AC-1).
- **Bundled shell pages use DESIGN.md token values** (static HTML/CSS, dark-first, no JS framework) — they are product-visible surfaces and follow the design grammar, but are not `web/` code and import nothing from it.
- **Readiness/timeout constants** — readiness poll deadline 30s (then `RuntimeStartFailed` with log tail), page-load deadline 10s (then error state, US-013.AC-2), crash-restart backoff 3 attempts max then terminal `Disconnected` (US-007.EC-2). Constants live in one Rust module; QA labs may not override them (no config — they are contract, not preference).
- **"First paint" is precisely the webview page-load-finished event** (Tauri `on_page_load` / `PageLoadEvent::Finished` on the window's webview) — a load guarantee, not a renderer-paint guarantee (N-002). The flash-free property (hidden window + theme background color until show) is a separate visual behavior verified by capture, not by this event.
- **Runtime-update staging order is fixed:** download + verify complete **before** the daemon is quiesced or stopped; stop→swap→start is the only interval inside the update lock. IT-017 asserts exactly this single guarantee (N-003) — there is no "restore after stop" alternative path.
- **Shell states render in a TWO-WINDOW shape** (analysis 06 §3, current Tauri docs): a bundled `boot` window (IPC-capable, CSP-hardened, owns loading/provisioning/error/skew/update screens) plus the `main` window created hidden at the daemon origin and shown only after the bound handshake passes; `boot` closes on success and reopens for disconnected/recovery states. No URL-swapping of one window across origins (webview history could walk back into shell pages; storage buckets split). ADR-007's navigation fencing applies to `main`; `boot` never navigates to remote origins.
- **Single durable identity + compiled channel default** (analysis 07 §6): one `identifier` `com.compozy.os`, one product name across channels — per-channel identity's data-isolation benefit is illusory for a thin shell (state lives in the daemon; two identifiers would mean two "single instances" attached to ONE daemon). The channel is a compiled-in constant (`option_env!` pattern; unset in dev builds ⇒ update checks disabled entirely), overridable only by a persisted setting that stays hidden until stable graduation (US-018.AC-2 upheld: nothing selectable in v1). Graduation protocol: publish the first stable version to BOTH feeds so beta users roll forward and switchers no-op at equal version — activation with a zero-width gap, no downgrade, BR-10 never engaged.
- **Update-check ordering is a safety property:** the app-update check runs **before** daemon resolution/startup on every launch (analysis 07 risks — a crash-on-launch build whose updater never runs is fleet-unrecoverable under BR-10, tauri#12720); daemon-start failure degrades to the boot window's error state, never a crash.
- **Updater hardening baked into the design** (analysis 06 §4, 07 §1): macOS — the shell takes its own `.app` backup before `install()` and restores it on failure (upstream #3505 can delete the app), and post-update QA asserts installed perms are not `0700` (#3506); Windows — consent is collected before install starts and intent is persisted in `on_before_exit` because in every released plugin version `downloadAndInstall` never resolves (#3514; re-pin when the fix ships); a feed `404`/`ReleaseNotFound` is surfaced as a feed-error state with `last_checked_at`, never conflated with "up to date" (US-015.AC-2); the updater builder wires a **two-pubkey rotation fallback** (check with the current key, retry once with the previous) from release one.
- **Feed/payload layout + publish atomicity** (analysis 07 §4): channel manifests at `desktop/<channel>/latest.json` + `desktop/<channel>/runtime.json(+.minisig)` (no-cache); immutable payloads under `desktop/v/<version>/…` (+ runtime archives under `desktop/v/runtime/<version>/…`) with `immutable` cache headers; publish order is payloads-first, manifest-last (single atomic PutObject); post-publish verification re-fetches the live feed and re-verifies one payload signature. Release gates additionally assert the shipped binary keeps the **default strictly-greater comparator** (no `allowDowngrades`, BR-10 regression guard) and that every published version is **strictly greater than the live feed version** (an equal-or-lower roll-forward would be invisible fleet-wide).
- **No fallback / no compat shim / no placeholder:** the shell ships no offline provisioning fallback, no alternate unauthenticated feed, no "temporary" unsigned build path, and no compatibility branch for pre-desktop provenance-marker absence (absence simply means "not app-owned"). If a decision here changes, the spec changes — not the code alone.

### Known Risks

- **SSE concurrency under OS webviews (top risk, now quantified):** the web app opens **eight independent SSE stream families plus one WebSocket** (home activity, session catalog, per-session transcript, task, task-run conversation, bridge health, extension logs, loop runs; window-manager WS) — instantiated per view, so stream-heavy layouts can exceed the ~6 per-origin HTTP/1.1 connection limit on WKWebView/WebView2/WebKitGTK; Electron references needed engine flags with no Tauri equivalent. Stream tickets never fire on localhost (the client latches `mode="local"`), so the shell adds no auth dimension to this. Mitigation: measured validation on real OSes under streaming load is a **release gate** (PRD constraint, E2E-021); if starvation is confirmed, the escalation is a daemon-side stream multiplexing/h2c initiative — explicitly not a shell proxy (ADR-007).
- **WebKitGTK rendering quirks (Linux):** NVIDIA/compositing blank-window issues are documented ecosystem-wide; remediation env vars are documented in the support runbook, not auto-applied.
- **Signing/notarization pipeline fragility:** first-release path depends on remapped legacy credential names (ADR-004); mitigated by a draft-release rehearsal before the first public ship and fail-closed gates (US-021).
- **Updater key custody:** single `pubkey`, no rotation support upstream (tauri#7585) — lost key strands the fleet. Mitigated by the custody runbook launch gate (generation ceremony, offline backup, rotation-via-transitional-release procedure) per ADR-004.
- **Feed single-point:** one malformed platform entry breaks updates for all platforms — mitigated by CI schema validation + post-publish verify (US-021.AC-2).
- **Rollback story (decided within PRD BR-10):** downgrades are never offered (US-015.EC-2 ignores an older feed), and `allowDowngrades` is deliberately **not** enabled — the emergency lever is **roll-forward**: re-publish the last good build's content under a new higher version. This keeps the updater's comparator strict, needs no v1 config, and works for every installed client forever; the release runbook documents the roll-forward procedure.
- **Provenance drift:** users manually replacing the app-owned binary make ownership inconclusive → detection falls back to recommendation-only (BR-8); status names the binary path so support can diagnose.

## Extensibility Integration Plan

- **Extension manifests, hooks, skills/capabilities, tools/resources, registries, bridge SDKs, MCP sidecars, protocol docs:** none added, changed, or removed. The shell is a client of the daemon's existing HTTP surface; extensions reach the daemon identically with or without the app in front (verified surfaces: extension host API, hook dispatch, provide surface set, bridge SDK contracts — all daemon-side, untouched).
- **New extension points:** deliberately none in v1 (PRD constraint); shell-level extension points are a future decision requiring their own spec.
- **Official Compozy skill (`skills/compozy/`):** gains the desktop surface at ship — install paths, `compozy app open|status`, update semantics, ownership rules (Impact Analysis row; sequencing step 9).

## Agent Manageability Plan

- **CLI:** `compozy app open [path] | status | update [--check|--apply <app|runtime>] | retry | diagnose` with `-o human|json|toon`, schema-versioned output, deterministic error identities (`app_not_installed`, `app_not_running`, `app_launch_failed`, `invalid_target_path`, `app_state_schema_unknown`, `app_control_unavailable`). Agents branch on `state`, `error.code`, and `update.*` vocabularies (stable, enumerated above). Every capability the shell UI has — including provisioning retry, update consent/apply, and diagnostics — is agent-operable (B-006); UI and CLI call the same control-socket primitive.
- **HTTP/UDS:** the only daemon contract change is the quiesce readout (closed end-to-end with codegen). App-local state deliberately does not become a daemon route — parity means "CLI reads the same schema-validated file and drives the same control socket the shell UI uses". Runtime-side truth agents need (versions, update availability, managed state, drain state) has HTTP/UDS parity via `/api/status`, `/api/settings/update`, and the drain surface.
- **Status/config discovery:** `[app]` keys are visible through the standard config surfaces (`compozy config`, generated docs); `app.json` and `app.sock` schemas are documented with the CLI reference; the canonical schema file ships in-repo (`desktop/schema/`).
- **Deterministic errors:** enumerated in Core Interfaces; every shell error state carries a stable `ShellErrorCode` + bounded `safe_message` that `app.json`/`compozy app status`/`diagnose` expose verbatim (redaction-gated, B-012).
- **Remote operation:** `compozy app` operates the local machine's shell (the desktop is a local-machine surface); remote daemon management flows stay on the existing remote-gateway CLI profiles, unaffected.

## Config Lifecycle

- **Added keys:** `[app] update_check` (bool, default `true`), `[app] update_check_interval` (duration string, default `"6h"`, bounds `[15m, 168h]`).
- **Structs/defaults/validation:** `AppConfig` in `internal/config` with defaults in `DefaultWithHome` and bound validation errors naming the exact key path; merge/overlay behavior identical to every other section (no special cases).
- **Consumers:** the Rust shell reads the file directly. Policy (single authority, ADR-010 amended): **absent** file/section → defaults; **present but invalid** → the update consumer fails closed (no network checks) with a stable `ConfigInvalid` diagnostic in `app.json`/status — the shell never "defaults through" configuration the canonical Go loader would reject. Key names, defaults, bounds, and the validation corpus are shared fixtures both languages test against (drift is a build failure).
- **Examples/docs/tests:** `config.toml` example gains the section; the configuration reference at `packages/site/content/docs/configuration/config-toml.mdx` is **hand-authored** (not generated) and gains the `[app]` section in the same change; validation tests extend the config package's canonical suite. The root `Config` struct lives in `internal/config/config_extensions_sandbox.go`; `AppConfig` follows the established per-section file pattern (own struct file, `defaultAppConfig()`, `Validate()` with `ValidationError{Path, Message}`). Removed/renamed keys: none.
- **Not config:** channel (build identity, ADR-005/010), window geometry (plugin state), supervision timeouts (contract constants).

## Web/Docs Impact

- **`web/`:** the shell requires zero web changes (ADR-007 property, unchanged). The absorbed brand cut edits **display strings only** (product-language "Compozy" → "CompozyOS"); brand marks route through the designer lane and the `@compozy/ui` inventory — no behavior, route, or API change.
- **`packages/site`:** installation page gains the desktop channel as the primary interactive install; new desktop getting-started page (first run, attach behavior, update experience, ownership rules, recovery state); **generated** CLI docs regenerate for the `compozy app` tree (`make codegen` → hidden `compozy doc` emits `packages/site/content/docs/cli/app/`) **plus the hand-maintained `cli/meta.json` navigation entry** (N-004); the **hand-authored** configuration reference (`configuration/config-toml.mdx`) gains the `[app]` section; site-wide product-language sweep (ADR-012); threat-model/runbook additions: desktop support runbook (log locations, WebKitGTK remediations, manual-download fallback, update recovery procedure, roll-forward release procedure).
- **Official skill routing:** `skills/compozy/` gains desktop-surface routing entries alongside the content update (N-004).
- **QA tracker:** new `untested` content-addressed scenarios — one per changed behavior, each mapped to its PRD story + test IDs with per-OS evidence requirements (N-004): install/first-run, attach-to-running, start-installed, app update, runtime update (both ownership branches), update recovery state, deep link (running + cold start), single-instance focus, quit contract, agent CLI journey — walked per `qa-execution` before workstream close.

## Delete Targets

The desktop capability itself is additive (CLI verb tree, config section, `InstallMethod` value, release jobs — all new, nothing replaced). The absorbed brand hard cut (ADR-012) has real delete targets, removed in the same sweep with no aliases or transition prose:

- `docs/_memory/glossary.md` — the "Compozy is the public product name…" definition (replaced by: CompozyOS is the product; `compozy` is its command).
- Every product-language "Compozy" occurrence in `packages/site` (marketing + docs prose), `COPY.md` rules that treat "Compozy" as the product name, `PRODUCT.md`, `web/` display strings, `skills/compozy/` prose, and README product prose (command examples and identifiers stay `compozy` by rule).
- `_brief.md` superseded decisions (annotated, not silently edited): quit-stops-owned-daemon wording, GitHub-Releases hot-path feed, `compozy://` scheme.
- ADR-006 decision 2 (superseded by ADR-012 — recorded in both files).

No compat shims exist anywhere: no dual naming, no alias scheme, no fallback feed, no legacy provenance reader (marker absence simply means "not app-owned").

## Compozy Impact Audit

- **Native tools:** no impact — checked `compozy__*` toolsets/descriptors/schema digests: the shell adds no tool and changes no daemon tool surface; `compozy app` is a CLI verb, not a native tool. The tool IDs keep the `compozy__` prefix by rule (command identifiers, ADR-012). Revisit only if a desktop-launch native tool is ever proposed.
- **Extensibility and hooks:** no extension/hook/registry/bridge/MCP contract changes (see Extensibility Integration Plan); config lifecycle impact is the `[app]` section; the bounded quiesce-readout contract change ripples through generated OpenAPI/TS only; official skill `skills/compozy/` update required at ship (content + routing + rename).
- **Workspace data isolation:** no new workspace-scoped datum. Shell files (`app.json`, `update.lock`, update journal, provenance marker, window state, logs, `app.sock`) are machine-local, home-scoped, and carry no workspace/session/agent data; all workspace data continues to flow exclusively through the daemon's existing HTTP/SSE surfaces with unchanged scoping. Checked: no new store/cache/SSE/event path exists.
- **Official Compozy skill:** update required at ship (new surface, new CLI verbs, update + recovery semantics, product-language rename) — sequenced in step 11.

## Architecture Decision Records

- [ADR-001: Thin shell over existing runtime and web UI](adrs/adr-001.md) — direction; zero web changes.
- [ADR-002: Attach-first runtime relationship with guided provisioning](adrs/adr-002.md) — resolution ladder; never disturb the unowned.
- [ADR-003: Update ownership split by installer](adrs/adr-003.md) — app self-updates; provisioned runtimes via app; managed installs get recommendations.
- [ADR-004: Three-platform launch, org signing reuse, own-domain hot update path](adrs/adr-004.md) — signing credentials; `releases.compozy.com`; custody gate.
- [ADR-005: Single beta channel, stable identity reserved](adrs/adr-005.md) — channel-scoped feed layout from day one.
- [ADR-006: CompozyOS durable identity; `compozy app` verb](adrs/adr-006.md) — naming; no CLI collision.
- [ADR-007: Window loads daemon-served UI; pure-Rust shell, zero page IPC](adrs/adr-007.md) — loading strategy; fixed-port origin; navigation fencing.
- [ADR-008: First-run provisioning by download to app-owned path + provenance marker](adrs/adr-008.md) — provisioning mechanics; ownership ground truth.
- [ADR-009: Deep links as validated product-path pass-through](adrs/adr-009.md) — link model; `compozy app open [path]`.
- [ADR-010: Minimal `[app]` config; shell reads config.toml directly](adrs/adr-010.md) — config lifecycle; channel is build identity; *amended round 1:* invalid config fails the update consumer closed.
- [ADR-011: App-side runtime-update apply; `desktop-app` install method](adrs/adr-011.md) — apply mechanics; managed-refusal preserved; *amended round 1:* signed runtime publication, mutation transaction, quiesce contract, migration-aware recovery (decisions 5–8).
- [ADR-012: Product-language hard cut to CompozyOS; command identifiers stay `compozy`](adrs/adr-012.md) — brand migration absorbed into this workstream; supersedes ADR-006 decision 2.

## Assumptions / Defaults

- **Security posture (stated, not silent):** the daemon's localhost HTTP surface is unauthenticated by design (browser-protection middleware + loopback guards only); the desktop app inherits this local posture unchanged and makes an always-running daemon more likely — accepted for v1; remote exposure remains exclusively the remote-gateway feature's authenticated domain. The shell adds no secrets: the updater public key is public, and no tokens or credentials exist shell-side.
- Version handshake defaults: the app pins a minimum supported runtime version (build constant) and treats an unknown/newer `schema_version` **date string** as newer-than-supported (both directions land in the guided skew state, BR-11). The comparison is against the real contract shape — `schema_version` is a date-string constant and daemon fields are nested under `daemon.*` (B-002). Caution (verified): the constant does **not** move on additive payload changes (the gateway field landed without a bump), so the minimum-version constant is the primary compatibility signal; the schema constant guards contract breaks only.
- `app.sock` is created 0600 in `$COMPOZY_HOME` (same trust domain as `daemon.sock`); its protocol is versioned and unknown verbs return a deterministic error — no capability negotiation in v1.
- Home resolution in the shell mirrors the runtime's `resolveHomeDir` rules exactly (env `COMPOZY_HOME` absolute-ized → `~/.compozy` default). The daemon/CLI's workspace-`.env` overlay (`ResolveHomePathsForWorkspace`) is deliberately **not** mirrored — a GUI shell has no workspace context; do not add it.
- Update check defaults: on launch (off the critical path) + every `update_check_interval` (default 6h); network failures are silent skips (US-014.EC-1).
- The runtime archive for provisioning matches the app's channel (beta) via `runtime.json`; there is no cross-channel provisioning.
- Dev builds use `compozyos-dev://` and a dev bundle identifier so installed and dev builds never contend (ADR-009).
- Baseline versions (Tauri 2.11.5, updater plugin 2.10.1, MSRV 1.77.2) verified live on 2026-08-09 (analysis 06). **Hard security floor: `tauri >= 2.11.1`** (CVE-2026-42184 local-origin confusion). Pin exact versions + commit `Cargo.lock`; never pin `tauri-plugin-deep-link` 2.4.8 (regression); re-pin `tauri-plugin-updater` the moment a release newer than 2.10.1 ships (carries the #3514 Windows fix).
- Deep-link platform reality (analysis 06 §5): macOS registration is static (Info.plist) and fires only for an installed app under `/Applications` — dev and QA walk installed builds; Linux calls `register_all()` on every launch (AppImage registration binds an absolute path). The scheme remains `compozyos://` (+`compozyos-dev://`) per ADR-009/012.
- Windows installer posture (analysis 07 §2): NSIS only, `installMode: currentUser`, updater `installMode: passive` — zero UAC prompts, standard users self-update; MSI (per-machine, IT-deployed) is a future artifact outside the in-app updater. Signing happens inside the bundler via object-form `signCommand` with Azure **Artifact Signing** (`artifact-signing-cli` — the service and CLI were renamed in 2026; the legacy signer script's tool name is stale).
- Conversation language BR-PT; every artifact, including this spec, English (SD-003).
