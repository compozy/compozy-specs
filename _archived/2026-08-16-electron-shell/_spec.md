# Spec: Electron Desktop Shell Migration

> **Technology-naming note (L-013 exception).** This spec is *about* replacing the desktop shell technology: Tauri, Electron, and the browser engines (WebKit, Chromium) are the product subject, so Part I names them. All other implementation vocabulary (languages, libraries, storage, wire formats) stays out of Part I.

---

# Part I — Product

## Overview

CompozyOS ships a desktop app whose only job is to render the runtime-served product UI in a native window and keep the runtime underneath it healthy. Today that shell is built on Tauri, which renders with the operating system's webview: WebKit on macOS and Linux — while every other surface of the product (development, browser access, the automated E2E suite) runs Chromium. The result is a structural quality gap: the surface the docs call "the primary interactive way to run CompozyOS" is the only surface rendering on an engine we never test, and it is the engine with the worst known behavior for our UI (the glass/blur depth effects hit unfixed upstream WebKitGTK compositing bugs on Linux; 11 of 14 desktop QA scenarios sit `blocked-verify` because no automation can drive the shell on macOS).

This program replaces the shell with Electron so the app renders on Chromium everywhere, and restructures where the shell's intelligence lives: runtime supervision and runtime self-update move to the runtime side, leaving the shell thin. The migration contract is **behavioral parity with three user-decided exceptions**: a single `compozy update` command replaces the two update verbs (the app-update verb is deleted, ADR-005), update offers move from the shell's overlay into the product UI — settings section plus a discreet menubar indicator, identical in a plain browser (ADR-006) — and self-update becomes headless-capable. Beyond those, the only user-visible changes are the rendering engine and the cutover itself.

**Who it is for:** developers on macOS and Linux using the desktop app daily; human or AI operators managing the app and runtime through the CLI without a GUI; the maintainer operating releases.

**Why it is valuable:** one engine means one UI truth — what passes in the browser is what ships in the app; the untestable becomes testable (the blocked QA queue unblocks); a whole class of Linux rendering failures and their documented workarounds disappears; and the update machinery consolidates into one mechanism the whole product shares.

## Goals

- **One engine, one UI truth.** The desktop app renders with the same engine as the browser surface on every supported platform; a UI behavior verified in the browser is the behavior in the app, including glass/blur depth effects on Linux without environment workarounds.
- **The primary surface becomes testable.** Automated end-to-end journeys drive the real installed app on macOS and Linux; all 14 `APP-*` QA scenarios are walkable and recorded `pass` on both platforms.
- **One update mechanism, one command.** Self-update is a single mechanism with one CLI entry point — `compozy update` covers runtime and app, and works headless with no desktop app installed.
- **Update state is visible where the operator lives.** Both update tracks are observable and appliable from the product UI (settings section + menubar indicator), with identical behavior in a plain browser — daemon truth, zero shell coupling in the SPA.
- **Byte-compatible manageability.** `compozy app status/open/retry/diagnose` and the deterministic error taxonomy survive unchanged; the published app-state schema takes one announced, lockstep version step (v1 → v2) so status can truthfully report the new update states — old/new mismatches keep failing with the existing deterministic schema-unknown error; `compozy app update` is the one deleted verb (ADR-005), announced in the cutover.
- **Zero legacy.** The old shell's code, build targets, CI jobs, signing inputs, helper scripts, and channel feed are deleted in the cutover — no bridges, no frozen artifacts.
- **Docs tell the truth.** Desktop docs describe the shipped shell; old-engine troubleshooting is deleted; a Chromium-first browser-support note is declared.

## User Stories

[Full user stories](_user_stories.md) — 29 stories, 3 personas.

- **Install & Boot** (US-001..US-006): fresh install/provision, in-place generational replacement, attach/start ladder, boot failure diagnostics, quit contract.
- **Window & Rendering** (US-007..US-012): single instance, geometry persistence, zoom, external-link routing, engine parity, UI crash recovery.
- **Deep Links** (US-013..US-014): running-app navigation, cold start, hostile-payload rejection.
- **Updates** (US-015..US-019, US-029): app update cycle and failure recovery, runtime updates with rollback, the single update command, configuration governance, web-visible update state.
- **CLI Manageability** (US-020..US-024): `status`, `open`, `retry`, `diagnose` preserved byte-compatible; `update` unified into the single `compozy update` command.
- **Release Operations** (US-025..US-027): new publish pipeline, feed repair, cutover execution.
- **Docs** (US-028): truthful desktop pages + Chromium-first note.

## Core Features

### 1. Chromium-rendering desktop shell

The app window renders the runtime-served product UI on Chromium on macOS and Linux. Functional requirements: boot window with real phase progress; main window against the running runtime; single instance; window-geometry persistence; bounded zoom with persistence; off-product links pushed to the system browser; UI crash recovery with bounded auto-reload; deep-link handling (running and cold-start) with hostile-payload rejection. Interaction: the shell is a client of the runtime's existing surfaces — it owns no product state.

### 2. Thin shell over a runtime-owned brain

Runtime supervision (attach → start installed → provision app-owned) and runtime self-update move to the runtime side; the shell orchestrates and displays. Functional requirements: the boot ladder's observable behavior is unchanged; supervision failures keep the explain-retry-logs contract; the shell↔runtime division is invisible to the operator. Interaction: enables feature 3 and shrinks feature 1's risk surface.

### 3. Unified self-update behind one command

One check/download/verify/swap/rollback mechanism for the runtime, one immediate-or-staged path for the app, and one CLI entry point: `compozy update` updates everything applicable on the host — the runtime always; the app when installed, immediately when running, staged for the next launch when closed. `compozy app update` is deleted (ADR-005). Functional requirements: staged and integrity-verified swaps; automatic rollback on failed health verification; single-flight per home with a deterministic `blocked` refusal; refusal of updates whose data-schema requirements exceed the installed app's supported range; identical observable outcomes whether triggered from the CLI or the product UI; multi-target structured results.

### 4. App self-update on the beta channel

The app track finds, downloads, verifies, and applies app updates; background checks are read-only; mutations are explicit — a product-UI action or `compozy update`. Functional requirements: failure detection on next boot with honest version truth; suppressed auto-checks after a failed attempt until recovery; resume/retry of interrupted downloads; strict version monotonicity; staged-on-next-launch semantics when applied while the app is closed.

### 5. Cutover and deletion

One release swaps generations: new artifacts and feeds live, old feed objects deleted, old shell code/CI/signing inputs deleted in the same change set, announcement with manual re-download and cleanup instructions. No bridge of any kind (ADR-003).

### 6. Docs and QA refresh

Desktop docs rewritten to the shipped shell; WebKitGTK troubleshooting deleted; Chromium-first browser note added; all `APP-*` scenarios reset to `untested` and walked to recorded verdicts on both platforms as the release gate, plus one real beta N→N+1 app auto-update cycle.

### 7. Web-visible update surface

Update state for both tracks becomes daemon truth rendered by the product UI (ADR-006): the settings Updates section presents runtime and app tracks with an apply action where self-apply is possible, and a discreet menubar indicator appears only while an update is available. The shell's boot overlay keeps only non-interactive phases — boot progress, applying progress, version skew, error — and its interactive offer is deleted. Rendering is identical in the app and a plain browser; the product UI gains zero shell-awareness. Surface inventory: `_uiux.md`.

## Business Rules

**Identity invariants**

1. Product name `CompozyOS`, install identity `com.compozy.os`, and deep-link scheme `compozyos://` never change across the migration.
2. New installers replace old-generation installs in place under the same identity.

**Runtime lifecycle**

3. Boot resolution order is exactly: attach to a healthy running runtime → start the installed runtime → provision an app-owned runtime.
4. At most one runtime process per home; the app never duplicates a healthy runtime.
5. The app refuses to attach to a runtime below its minimum supported version, with an explained repair path.
6. Quitting the app never stops the runtime. OS-forced exit gives the same guarantee.
7. Every boot failure names its phase and cause and offers retry, open-logs, and quit; retries are bounded with backoff — never an unbounded loop, never a silent hang.
8. A retry against a healthy app is a no-op and must never corrupt recorded state.

**Updates**

9. Only the `beta` channel publishes; `stable` is reserved and refuses publication.
10. Version monotonicity: a client is never offered an equal or lower version; downgrades are refused.
11. Background checks are read-only; downloads and installs happen only on explicit action — a product-UI apply or `compozy update`.
12. Default check cadence is 6h, configurable within 15m–168h via the existing `[app]` keys; `update_check = false` disables background checks for both tracks.
13. App-track and runtime-track updates are mutually exclusive and single-flight per home; concurrent requests receive the deterministic `blocked` refusal naming the current holder.
14. Runtime updates are staged, integrity-verified before swap, health-verified after swap, and roll back automatically on failure; interruption at any point resolves to a working runtime on next boot.
15. A runtime update whose data-schema requirements exceed the installed app's supported range is refused with a compatibility explanation.
16. The update mechanism mutates only installs it owns; package-manager-managed installs receive the exact upgrade recommendation and are never mutated.
17. Artifacts are signed and verified before install on every path; a verification failure leaves the current install untouched.
18. Publication ordering: payloads become available before channel manifests flip.
19. `compozy update` is the only update command: a default invocation covers every applicable target on the host; the app track applies immediately when the app is running and stages for the next launch when it is closed. `compozy app update` does not exist — no alias, no shim.
20. One decision surface: update offers render only in the product UI (settings Updates section + menubar indicator, both bound to daemon truth). The shell overlay never offers an update; it reports only non-interactive phases — boot progress, applying progress, version skew, error.
21. The menubar indicator appears only while an update is available; it never renders progress, counts, or errors.

**Deep links and navigation**

22. A deep-link target must be an absolute product path; links carrying hosts, traversal segments, backslashes, embedded schemes, or external URLs are rejected and land the app on the default view.
23. Off-product links open in the system browser only when they parse as safe web links; other schemes are never executed by the app window.
24. A deep link arriving before the UI is ready is queued and applied exactly once.

**Control surface**

25. The local control channel is per-user and local-only; its schema version and the app-state schema govern compatibility, and unknown versions produce the deterministic schema-unknown error.
26. The deterministic error taxonomy is frozen: `app_not_installed`, `app_not_running`, `app_launch_failed`, `invalid_target_path`, `app_state_schema_unknown`, `app_control_unavailable`, `diagnostic_consent_required`.
27. Diagnostic bundles require explicit consent; no diagnostic output ever contains secrets.

**Cutover**

28. No bridge: the old channel feed is deleted, not frozen; abandoned installs are permitted to log failed checks.
29. The cutover change set deletes every old-shell code path, build target, CI job, signing input, and helper script — a surviving reference is a defect of the cutover, not a follow-up.

## User Experience

**Journeys** (each maps to an existing QA journey and is preserved verbatim):

- **First run** (`J-desktop-first-run`): download → install → open → boot window with phase progress → product UI. No terminal required.
- **Daily attach** (`J-desktop-attach-daily`): open app → instant attach to running runtime → work → quit → runtime keeps serving CLI/browser.
- **Update moment** (`J-desktop-update-moment`): background check surfaces an update → operator applies → progress → restart (app) or quiesce-swap-verify (runtime) → back to work on the new version; failure lands on honest status with a retry.
- **Link-driven** (`J-desktop-link-driven`): a `compozyos://` link from the terminal focuses or cold-starts the app on the exact destination.
- **Headless agent** (`J-desktop-agent-headless`): an agent runs `compozy app status`, `compozy update`, and `compozy app diagnose` over structured output, never touching a GUI.

**Accessibility:** full keyboard operability of shell chrome (menus, zoom, quit); zoom bounds respect low-vision use; failure dialogs are reachable and readable without pointer precision. The product UI's own accessibility floor (PRODUCT.md) is unchanged.

**Onboarding and discoverability:** unchanged — the install pages and getting-started flow remain the entry points; the announcement covers the one-time manual re-download for existing installs.

## High-Level Technical Constraints

- **The runtime's contracts are fixed.** The app is a client of existing runtime surfaces and the existing home layout; the migration may not change runtime APIs or state locations. `compozy app status/open/retry/diagnose` and the error taxonomy are byte-compatible; the app-state schema takes one explicit, enumerated version step (v1 → v2, shipped lockstep across shell and CLI) — the designed evolution path, not silent drift; the single deleted verb is ADR-005's decided break.
- **Version-compatibility handshake preserved:** the app declares a minimum supported runtime; runtime updates carry data-schema requirements the app checks before applying (rules 5 and 15).
- **Signed everything:** artifacts signed, feeds authenticated, macOS notarized; distribution consolidates on the same public release channel the runtime already updates from (ADR-007), and the custom distribution origin is retired.
- **Performance from the user's seat:** boot-to-UI no slower than the current shell on the same hardware; window interactions at native fluidity; installer size grows (a shell-embedded engine is ~5-7× the current download) — an accepted, announced trade-off.
- **Privacy:** no telemetry added; diagnostics remain consent-gated (rule 24).
- **Engine update treadmill:** the embedded engine receives upstream security releases on a fixed ~2-month cadence with a bounded support window; the program must leave the team able to absorb roughly one platform bump per quarter as routine maintenance.
- **Agent/operator manageability outcome:** the full app lifecycle — inspect, open, update both tracks, retry, diagnose, export evidence — is operable through CLI structured output with deterministic errors, GUI-free; runtime self-update additionally works with no app installed.
- **Extension ecosystem expectation: none.** The shell is not an extension surface: no extension manifest, hook, skill, tool, resource, or registry references the desktop shell today, and this program adds no extension points to it. Extensions keep interacting with the runtime, never the shell.

## Non-Goals (Out of Scope)

- **Windows support.** Blocked externally (signing account); the design must not preclude it, but no Windows target ships in this program.
- **New desktop features.** No tray, native notifications, global shortcuts, auto-launch at login, or multi-window — strict parity is the contract (user decision Q4).
- **Migration bridge.** No final old-generation release, no automatic cross-generation update, no frozen feed (ADR-003).
- **Safari/WebKit browser parity work.** The browser surface stays as-is; Chromium-first is declared, other browsers are best-effort (user decision Q5).
- **Web UI changes beyond the update surface.** The only web changes are ADR-006's update surface (settings Updates section + menubar indicator); no other product UI features or design changes ride along, and the SPA gains no shell-awareness.
- **Rebranding.** Name, identity, and scheme are frozen (rule 1).

## Open Questions

None. Every product fork raised in Stage 1 was resolved: scope (ADR-002), platforms, installed-base path (ADR-003), parity bar, docs claim, release gate, runtime-update unification (ADR-004), cutover defaults (ADR-003).

---

# Part II — Technical

## Executive Summary

The shell swap rides on one structural insight: the runtime and the CLI are the same binary, and `internal/update` already implements a hardened self-update (sigstore/TUF verification, checksum catalog, artifact-size policy, backup-restore, daemon-restart coordination). The program therefore does not port the Rust updater to the new shell — it extends `internal/update` into the single mechanism (ADR-004) behind the single command (ADR-005), relocates update/supervision intelligence to the Go side (ADR-002), and reduces the shell to commodity subsystems with direct reference implementations in `.resources/{synara,hermes,t3code}` (see `analysis/electron-reference-blueprint.md`). The Electron shell (`desktop/`, TypeScript, Bun workspace member) renders the daemon-served SPA directly (`loadURL(daemonOrigin)`), keeps a boot window for non-interactive phases, and owns only: windowing, deep links, single instance, launching the **bundled** runtime's Go bootstrap (the lockstep `compozy` binary ships inside the app package, so the boot ladder is Go-owned even on an empty home — B-010), rendering Go-owned supervision and update progress, executing app-track installs, and a reduced, hardened control server. Runtime supervision (start, readiness, retry policy, failure classification) is Go-owned and shell-rendered — the shell's own logic ends at "download a verified binary, then drive the Go supervisor" (honoring ADR-002 literally). Every mutation of the runtime install is owned by a durable per-home Update Operation executed by a detached coordinator that survives daemon restarts (ADR-009). Update state becomes daemon truth served by the extended settings surface and rendered by the web UI (ADR-006). Artifacts publish per-arch to GitHub Releases in lockstep with the CLI (ADR-007/ADR-008); `releases.compozy.com` and all Tauri infrastructure are deleted at cutover (ADR-003).

## MVP Boundary

MVP boundary: the numbered tasks produced by `cy-create-tasks` implement phases P1–P4 below — runtime-side update unification with the Update Operation coordinator, compatibility gate, and daemon-served CSP (P1), the web update surface (P2), the Electron shell at parity (P3), and the release pipeline with the publish/repair authority + cutover (P4) — plus the trailing `qa-report`/`qa-execution` pair. Post-MVP, explicitly out of scope per Part I Non-Goals: the Windows target, tray/notifications/global shortcuts/auto-launch, any migration bridge, Safari-parity work, and web UI changes beyond the update surface.

## Developer Experience

- [Developer experience contract](_dx.md) — CLI (`compozy update` multi-target; `compozy app status/open/retry/diagnose`; deleted `compozy app update`), deep links, `config.toml`, deterministic errors, overlay parity reference.
- [UI change map](_uiux.md) — S1 settings Updates section (two tracks + apply), S2 menubar update indicator.

## System Architecture

| Component | Responsibility | Boundary |
| --- | --- | --- |
| `internal/update` (extended) | Unified self-update library: multi-target check with verified release identity, Update Operation journal (acquire/transition/recover), runtime apply for every non-managed install method (now including `desktop-app`), provenance ownership | Pure library; imported by `internal/cli` and `internal/daemon` (unchanged importers) |
| Update coordinator (ADR-009) | The sole executor of runtime mutations: verify → swap → restart daemon → health-check → finalize/restore, across daemon generations; runs foreground under `compozy update`, or detached when spawned by the daemon apply endpoint | `internal/update` logic behind an internal `compozy` entry point; ownership lives in the operation record, never in a process lifetime |
| Runtime supervision (Go, bundled) | The full boot ladder from an empty home: resolution (attach → start → provision), install-from-bundle self-provision, start, readiness probing, bounded retry/backoff, give-up classification, typed phase progress. The lockstep `compozy` binary ships **inside the app package** (per-arch extra resource) with a build-embedded digest manifest; the shell executes the bundled bootstrap entry point (`compozy daemon bootstrap -o jsonl`, internal plumbing) from t=0. **Trust root (B-027)**: the pre-execution integrity decision belongs to the layer that runs *before* the payload — on macOS the signed/notarized app package covers the bundled binary (a tampered bundle fails the platform signature), and the shell additionally verifies the payload digest against its build-embedded manifest **before first spawn** on every platform (corruption detection; on Linux, package authenticity roots in the download checksums, stated honestly). The bundled binary never self-certifies | Policy in Go with unit-tested tables; the shell owns pre-spawn digest verification + spawn-failure rendering only; Go owns everything after spawn (ADR-002 honored; B-001/B-010/B-027) |
| Settings update surface (`internal/api/core` + transports) | `GET /api/settings/update` extended to both tracks + live operation lifecycle; new `POST /api/settings/update/apply` returning `accepted` + operation id; HTTP+UDS parity | Handlers as `BaseHandlers` methods; one-line registration per transport |
| `compozy update` (CLI) | The single command: acquires the Update Operation, runs the coordinator in the foreground, reports the multi-target record with aggregate status | `internal/cli/update.go` rewritten around `MultiState` |
| Web update surface | `SettingsUpdateTrackRow` ×2 + `MenubarUpdateIndicator`, per `_uiux.md`; daemon truth via existing settings query options, faster poll while an operation is live | `web/src/systems/settings`, `web/src/systems/os` |
| Electron shell (`desktop/`) | Window/boot UI, single instance, deep links, external-link guard, zoom, window-state, launching the bundled Go bootstrap and rendering its typed progress, app-track updater **execution** (bound to the operation record's verified asset identity), hardened control server (`navigate`/`retry`/`diagnose`/`export_diagnostics`). The shell reads **no update configuration** — the daemon is the sole `[app]` cadence consumer (B-015) | TypeScript main process, t3code-style module layout (500-line cap per file); product window has **no preload** |
| Publish/repair authority | `cmd/compozy-desktop-release publish` and `repair`: idempotent payload upload with digest checks, exact-inventory verification, channel-manifest flip last, last-known-good selection, audit records | Go tool; imported package `internal/desktoprelease` (inventory/version policy + channel authority) |
| Release pipeline | Per-arch electron-builder builds, mac notarization, manifest merge, lockstep GitHub Release publish via the authority above, packaged-app smoke | CI workflows under a single channel concurrency group |

Data flow: shell `loadURL(http://127.0.0.1:<port>)` → daemon-served SPA (unchanged path, `internal/api/httpapi/static.go`, now with a CSP header). Update reads: web/CLI → settings surface → `internal/update` projection incl. the live operation record. Runtime apply: any surface acquires the Update Operation; the coordinator executes it (foreground for CLI, detached for the daemon endpoint) and journals every phase. App apply: the operation record's app section is the request — the running shell observes the record (fs watch + poll fallback), transitions `staged → applying` under the record's discipline, drives the app updater against the recorded asset digest, and writes the post-restart verification; a closed shell resolves the record on next launch. Boot-time daemon recovery and launch-time shell recovery resolve any non-terminal record by phase.

Content-loading decision: direct `loadURL` of the daemon origin (today's model) — maximum browser/app parity, zero new moving parts; the custom-scheme reverse proxy from t3code (`.resources/t3code/apps/desktop/src/electron/ElectronProtocol.ts`) is the recorded fallback if origin stability across port changes ever becomes a real problem.

## Architectural Boundaries

- `internal/update` stays a leaf library imported only by `internal/cli` and `internal/daemon` (status quo). It gains no importers and imports no api/cli/daemon package.
- The settings surface extension lives in existing packages (`internal/api/core`, `internal/api/httpapi`, `internal/api/udsapi`); **no new `internal/api/*` subpackage**, so `magefiles/boundaries.go` is unchanged.
- `internal/daemon` remains the sole composition root: the extended update controller wires in `boot_settings.go` exactly like today's `settingsUpdateController`.
- `internal/desktoprelease` shrinks (feed generation and Tauri channel-config deleted) and remains imported only by `cmd/compozy-desktop-release`.
- `desktop/` becomes the Electron app (Bun workspace member); `desktop/schema/` is untouched and keeps its three consumers (Go embed, shell, fixtures).
- No Go package imports `desktop/` beyond the existing `desktop/schema` embed; the shell talks to the runtime only through public surfaces (HTTP status, spawn, home files).

## Implementation Design

### Core Interfaces

Concrete types live in `internal/update`; consumers define the narrow interfaces they use (interfaces-where-consumed, N-001):

```go
// internal/update — concrete types (provider side; no exported god-interface).
type Target string // "runtime" | "app"

// MultiState is the LIVE projection backing GET /api/settings/update and the
// app-status update block. The CLI's TERMINAL record is a distinct, explicitly
// mapped projection (below) — three contracts, one source, no divergence (B-014).
type MultiState struct {
    Aggregate Status           // live precedence: failed > blocked > applying > staged > updated > available > up-to-date
    Operation *OperationView   // nil ⇔ no live Update Operation on this home
    Runtime   RuntimeTrackState
    App       *AppTrackState   // nil ⇔ no desktop app installed on this host
}

type RuntimeTrackState struct {
    Status          Status // up-to-date | available | applying | updated | failed | blocked
    InstallMethod   string
    Managed         bool
    CurrentVersion  string
    LatestVersion   string
    ReleaseURL      string
    Recommendation  string // managed installs: the exact upgrade command
    RestoredVersion string // set when the last apply rolled back
    DaemonRestarted bool   // frozen `_dx.md` field — preserved in every projection
    Message         string // frozen `_dx.md` field — preserved in every projection
    LastError       string
}

type AppTrackState struct {
    Status         Status // up-to-date | available | accepted | staged | applying | updated | failed | blocked | unsupported
    Running        bool
    CurrentVersion string
    LatestVersion  string
    ReleaseURL     string
    AttemptID      string
    LastError      string
    Message        string
}

// OperationView is the read projection of the durable Update Operation (ADR-009).
type OperationView struct {
    ID           string
    Revision     int64          // monotonic record revision — the CAS fence (B-021)
    Targets      []Target       // execution always runtime-first (B-020)
    ActiveTarget Target         // which track Phase/Percent describe; empty when waiting
    Phase        OperationPhase
    Percent      int            // -1 when the phase has no measurable progress
    Holder       *Holder        // nil while dormant (N-010); live = {PID, PIDStartTime, Surface, ExecutorGeneration, LeaseExpiresAt}
    Waiting      WaitingState   // "" | "waiting-for-app" — dormant operations are joinable, not blocking
    StartedAt    time.Time
    LastError    string
}
```

**Projection mapping (B-014/B-023).** (1) *CLI terminal record* (`compozy update`): the frozen `_dx.md` shape — per-target objects with `daemon_restarted` and `message`, aggregate precedence exactly `failed > blocked > staged > updated > available > up-to-date` (a terminal record never reports `applying`). (2) *Live settings projection* (GET): the full `MultiState` — including `operation.revision/active_target/waiting`, nullable holder, and the runtime track's `daemon_restarted` + `message` (no field of any projection is dropped by another transport). (3) *App-status projection* (`compozy app status`, schema v2): the `update` block carries `operation_id`, `phase`, `percent`. **Phase→UI mapping is exhaustive and fixed**: runtime `downloading→download` (percent from the download step), `verifying→verify`, `swapping→install`, `restarting→start`, `health-checking→ready-check`, `finalized→ready`; app `applying→download/verify` (percent from the updater's progress events), `installer-handoff→install`, `restarted→start`, `verified→ready` — `_uiux.md` S1 renders exactly these named phases; nothing is invented per surface. One computation feeds all three; each mapping is golden-tested against its surface contract.

**Typed vocabularies (N-005).** `Status`, `OperationPhase`, `Target`, and `Holder.Surface` are closed Go enums validated at every contract boundary (journal read/write, API encode/decode, CLI record) — an invalid phase, actor, or target cannot enter the journal or the generated API.

```go
// internal/cli — narrow consumer interface (extends today's updateManager shape).
type updateRunner interface {
    CheckAll(ctx context.Context, opts compozyupdate.CheckOptions) (compozyupdate.MultiState, *compozyupdate.Release, error)
    AcquireOperation(ctx context.Context, req compozyupdate.OperationRequest) (*compozyupdate.Operation, error)
    RunCoordinator(ctx context.Context, op *compozyupdate.Operation) (compozyupdate.MultiState, error)
}

// internal/daemon — narrow consumer interface behind core.SettingsUpdateController.
type settingsUpdater interface {
    Snapshot(ctx context.Context) (compozyupdate.MultiState, error)
    AcquireOperation(ctx context.Context, req compozyupdate.OperationRequest) (*compozyupdate.Operation, error)
    SpawnCoordinatorDetached(op *compozyupdate.Operation) error
}
```

```go
// internal/api/core — settings update controller, extended.
// Typed vocabularies end-to-end (N-005/N-009): raw strings exist only at JSON
// decode, validated into these types before any logic.
type SettingsUpdateController interface {
    GetUpdate(ctx context.Context) (SettingsUpdateStatus, error)                       // full MultiState projection
    ApplyUpdate(ctx context.Context, target compozyupdate.Target) (SettingsUpdateApply, error)
    CancelUpdate(ctx context.Context) (SettingsUpdateCancel, error)
}

type SettingsUpdateApply struct {
    Target      compozyupdate.Target      `json:"target"`
    Status      compozyupdate.ApplyStatus `json:"status"`       // accepted | blocked | failed
    OperationID string                    `json:"operation_id"` // set when accepted
    Message     string                    `json:"message"`
}
```

Apply is **asynchronous and truthful**: the endpoint returns `accepted` + operation id after the operation record is durably acquired — never a terminal verdict it cannot yet know (B-005/B-006). Terminal truth arrives through `GET /api/settings/update` (operation view + per-track states) once the coordinator or the shell journals it. The daemon endpoint spawns the coordinator **detached as a separate process** — the executor's lifetime is never the daemon's own (B-002); the daemon merely acquires, spawns, and serves the journal.

### Data Models

No SQLite change anywhere in this program. Update state is a computed projection over durable flat files with disciplined ownership:

```
~/.compozy/update-operation.json         — THE Update Operation (ADR-009): exclusivity + journal.
    schema_version      INT   — record contract version; unknown → refuse with diagnostics
    operation_id        TEXT  — correlates CLI/HTTP/UDS/web/events; fencing key for every transition
    requested_by        TEXT  — "cli" | "daemon" | "web" (via daemon)
    revision            INT   — monotonic record revision (B-021): every transition carries the
                                expected revision — the unambiguous CAS fence across both tracks
    targets             LIST  — ["runtime"] | ["runtime","app"] | ["app"]; execution is ALWAYS
                                runtime-first (B-020 — compatibility is a publication invariant)
    runtime.from/to     TEXT  — versions; release_tag, asset, digest — identity verified at acquisition
    runtime.backup_path TEXT  — restore source; empty after finalize
    runtime.phase       TEXT  — pending|downloading|verifying|swapping|restarting|health-checking|finalized|rolled-back|failed
    app.from/to         TEXT  — versions; asset, digest — the SAME verified identity the shell must install
    app.attempt_id      TEXT  — durable attempt correlation (US-016)
    app.phase           TEXT  — pending|staged|applying|installer-handoff|restarted|verified|failed
                                (installer-handoff = quitAndInstall → external installer window:
                                 NON-cancelable, blocks all other mutation until post-restart
                                 verification or its phase watchdog decides — B-022)
    app.consecutive_failures INT — needs-attention escalation counter
    app.watchdog_deadline    TS  — silent-failure detection bound (per-phase deadlines for
                                    applying and installer-handoff)
    holder               OBJ|null — executor LEASE (B-022): null while dormant; live =
                                {pid, pid_start_time, surface, executor_generation, lease_expires_at};
                                executor_generation is a random fence with bounded renewal cadence —
                                checked with the expected revision before EVERY transition and every
                                irreversible side effect, so an expired-then-delayed executor can
                                never act after a replacement takes over
    waiting              TEXT — "" | "waiting-for-app" — dormant, joinable state (holder null)
    deadline            TS    — operation-level watchdog for recovery sweeps
    last_error          TEXT  — safe message; never secrets
    DISCIPLINE (B-021/B-022): acquisition AND every mutation are serialized by the stable
    companion lock below; create-if-absent happens under that lock with a fully written,
    fsynced temp record atomically published at the final path — no reader ever observes
    a partial record.  All transitions go through ONE Go transition API —
    Transition(operation_id, executor_generation, expected_revision, transition) — and the
    shell NEVER writes or flocks the journal itself: its transitions execute the same API
    via the installed `compozy` binary (single implementation; no cross-language peer
    authority).  A new request finding a dormant record for the same target RESUMES it —
    the single-winner resume transition; dead/expired leases are recovered on acquisition,
    by a bounded periodic daemon sweep, and on shell launch, each via the same API.
    The per-phase recovery/cancel matrix is part of this contract:
      runtime pending|downloading|verifying → safe restart of the step; cancel allowed (dormant only)
      runtime swapping|restarting|health-checking → resume-or-restore per journal; cancel refused
      runtime finalized|rolled-back|failed → archive; cancel moot
      app pending|staged → resumable; cancel allowed
      app applying → watchdog decides (fail + re-offer); cancel refused
      app installer-handoff → NON-cancelable; post-restart verify or phase watchdog decides
      app restarted|verified|failed → archive path; cancel moot
    Terminal → idempotent (by operation_id) append to update-history, then record removal.
~/.compozy/update-operation.lock         — stable flock companion (never renamed): serializes every
    journal writer AND acquisition — coordinator, daemon sweep, CLI, and the shell's
    `compozy`-mediated transitions (B-021)
~/.compozy/logs/update-history.jsonl     — bounded append-only terminal-operation archive (audit + UT/IT evidence)
~/.compozy/bin/.desktop-provenance.json  — existing file; WRITER MOVES Rust → internal/update
    so install-method detection stays truthful after Go-side self-applies
~/.compozy/app.json                      — schema takes its ONE designed version step: v1 → v2
    (B-023 hard cut, announced): the byte-compatibility promise is explicitly amended —
    `schema_version` becomes 2; the `update` block's v2 shape is fully enumerated:
    `app_state`/`runtime_state` value sets extend with `staged`, `applying`, `blocked`,
    `accepted`; new properties `operation_id` TEXT, `phase` TEXT, `percent` INT.
    Shell and CLI ship the step lockstep; a v1 reader meeting v2 (or vice versa) fails
    with the EXISTING deterministic `app_state_schema_unknown` error — the evolution
    mechanism the contract was designed with. Schema file, fixtures, CLI validation,
    and both status projections update in the same change.
~/.compozy/update.lock                   — existing flock (internal/daemon/update_lock.go);
    retained as the short critical-section lock INSIDE the coordinator's swap step.
    The Operation record is the long-lived ownership layer above it — acquisition is
    creating the record, never observing the flock (B-003)
```

The pre-review `app-update-intent.json` concept is **deleted** — the operation record's app section replaces it (ADR-009).

Side-table-vs-JSON: does not arise — no new domain entity reaches a database; nothing here is matchable state. The flat files above are single-writer-per-phase records with atomic transitions, which is exactly the case JSON files are for.

### API Endpoints

- `GET /api/settings/update` (HTTP + UDS): the full `MultiState` projection — `{aggregate, operation:{id,revision,targets,active_target,phase,percent,holder|null,waiting,started_at,last_error}|null, runtime:{status,install_method,managed,current_version,latest_version,release_url,recommendation,restored_version,daemon_restarted,message,last_error}, app:{status,running,current_version,latest_version,release_url,attempt_id,last_error,message}|null}`. This is the complete data source `_uiux.md` S1 renders — named phases, percent, release links, recommendations, holder (null when dormant), rollback truth — nothing UI-invented, no field dropped relative to the Go structs (B-005/B-023, SD-007). Generated TypeScript regenerates (`make codegen`).
- `POST /api/settings/update/apply` `{target:"runtime"|"app"}` (HTTP + UDS): 200 with `SettingsUpdateApply` — `accepted` + `operation_id` after durable acquisition; deterministic `blocked` body naming the holder; `failed` only for acquisition-time failures; unknown target → 400 standard envelope. Terminal outcomes are read from GET, never invented at apply time. Registered in `internal/api/spec` for OpenAPI.
- `POST /api/settings/update/cancel` (HTTP + UDS): structured cancel for **dormant** operations only (`waiting-for-app` or expired lease) — cancels, archives with outcome `canceled`, and frees acquisition; a live executor lease declines with the holder (B-013). Mirrored by `compozy update --cancel`.
- Polling semantics: the web keeps the existing settings query cadence at rest and switches to a short interval (2s) while `operation` is non-null; no SSE stream is added in this program.
- All handlers are `BaseHandlers` methods; transports only register routes (`internal/api/{httpapi,udsapi}/routes.go`), per the no-transport-duplication rule.

### Release compatibility gate (B-004; publication invariant per B-020)

The invariant behind Part I rules 5 and 15 survives the `runtime.json` deletion in the new signed release contract:

- Each release carries a `compat.json` asset — `{runtime_version, min_app_version}` — listed in `checksums.txt`, therefore covered by the same sigstore verification as every artifact. `min_app_version` is the **sole** compatibility authority; data-schema compatibility belongs to the runtime's own migrations (the former `schema_heads` field is deleted — no unused contract surface, N-008).
- **Compatibility is enforced at publication** (B-020): the channel authority refuses any release whose `min_app_version` exceeds the previous channel generation's app version — a published runtime always accepts the app generation that can still stage its companion, so the frozen closed-app outcome (runtime updates now, app stages for next launch) is achievable for every published release, and requiring a newer app takes two releases by construction. The operation state machine stays **runtime-first, single-shape** — no app-first plan, no deferred runtime phase.
- The coordinator re-checks the gate **under the operation, immediately before the runtime mutation**, as a backstop: a runtime-only request targeting a release whose requirement the installed app does not meet journals `failed` with a compatibility explanation and mutates nothing (reachable only if the publication invariant was bypassed — e.g., a hand-built release).
- Attach-time is guarded in both directions: the shell keeps `MINIMUM_RUNTIME` (build-config sourced), and `/api/status` gains `min_app_version` so the shell refuses to attach when the running runtime requires a newer app.
- US-001.EC-4 and US-017.EC-1 keep their tests (restored in `_tests.md`); nothing about this gate is retired.

### Publish and repair authority (B-008, real linearization per B-024)

US-025/US-026 keep their full promise on the new distribution channel via `cmd/compozy-desktop-release`. Round-3 review verified the dependency contracts: the updater's release-list provider cannot express a channel pointer (it resolves prereleases from the repo feed, and the platform's "latest" flag excludes prereleases), so the design uses the **generic provider against a channel we actually own**:

- **Immutable generations**: every version's payloads live in that version's release and are never mutated after verification; per-arch manifests are generated per version and committed, never edited in place.
- **The channel is a git branch; the switch is a ref CAS**: channel manifests (`latest-mac.yml` merged per-arch, `latest-linux.yml`) live on a dedicated `channel-beta` branch; the app updater's generic provider reads their raw URLs pinned to that branch. `publish` = upload + verify payloads in the versioned release → commit the new manifest set (one commit, every platform file together) → **update the branch ref with the expected old SHA** — a provider-native compare-and-swap. One commit is atomic across all files; a lost CAS is a deterministic `channel_cas_conflict`, which serializes every entry point (CI job or direct CLI) with no lock asset and no TTL fencing gap.
- **Repair = ref CAS to a known-good commit**: history is the immutable, ordered audit trail; the known-good marker is a verified commit recorded in the audit metadata after post-flip verification; `repair` verifies that generation's payload inventory against the versioned release (refuse on any missing asset) and CAS-moves the ref. Interrupt anywhere → the branch still points at exactly one complete generation.
- **Idempotent audit identity**: each publish/repair writes one commit whose message carries the operation id — re-running an interrupted operation converges (same id ⇒ no duplicate flip).
- **Structured operator contract (N-006)**: both subcommands emit a JSON result (`{operation, channel_ref_before/after, verified_inventory, audit_commit, outcome}`) with deterministic error codes (`inventory_incomplete`, `channel_cas_conflict`, `verification_failed`); listed in the Agent Manageability Plan.
- **Real-provider rehearsal**: the E2E rehearsal exercises the actual generic-provider read path (raw branch URLs) and an interrupted publish at every step — not only a mock (B-024).
- The old shell scripts and `desktop-feed-repair.yml` are still delete targets; this authority is their replacement, not a survival.

### Shell control channel (wire-compatible subset, hardened)

`~/.compozy/app.sock`, newline-delimited JSON, `schema_version: 1`, 64 KiB cap — wire contract unchanged. Methods retained: `navigate`, `retry`, `diagnose`, `export_diagnostics`, `copy_diagnostics`. Methods deleted with their only caller (`compozy app update`): `update.check`, `update.apply`. `internal/cli/app_control.go` keeps its shape; the 2-minute extended timeout now applies only to `export_diagnostics`.

The reimplemented server carries an explicit security design (B-007, mechanism fixed per B-016): canonicalized non-symlink parent directory owned by the user at mode `0700`; socket at `0600`; **capability-token authentication** — the shell generates a random 32-byte token per app session, writes it to `~/.compozy/app.token` at `0600` (same-uid readability enforced by file mode), and every request carries it; the server compares timing-safe and refuses mismatches deterministically (`app_control_unavailable` cause `unauthorized`). Peer-credential syscalls were rejected as the mechanism because no supported Electron/Node API exposes `SO_PEERCRED`/`getpeereid` — the token design is implementable by the named owner without unaudited native bindings, and its lifecycle is explicit: rotated on every shell start, deleted on clean exit, never logged, never in diagnostics. Bounded concurrency (max connections, one in-flight request per connection, 64 KiB message cap, write backpressure); stale-socket cleanup only after a connect probe fails **and** ownership is verified. Adding the `token` request field is a lockstep CLI+shell change within `schema_version: 1`; a token-less request against the new server fails deterministically.

## Integration Points

- **GitHub Releases** (existing): release lookup and asset download via `internal/update`'s existing client, cache, and sigstore verification — unchanged trust chain, now also serving the app-track version truth (same release, lockstep tag).
- **electron-updater (GitHub provider)** in the shell for the app track: mac zip + per-arch `latest-mac*.yml` merged at release; `autoDownload=false`; the shell pins the install to the Update Operation's recorded release version and verifies the downloaded artifact digest against the operation's recorded identity before `quitAndInstall` — a feed change between Go-side check and shell-side install is a refusal, not a silent divergence (B-006). Known upstream pitfalls carried as pipeline guards (mac update-zip must preserve Electron Framework symlinks — `.resources/synara/scripts/lib/mac-update-zip.ts` is the reference).
- **`compat.json` release asset** (Release compatibility gate above): produced by the release pipeline from the runtime build, consumed by the coordinator and the shell's attach guard.
- **notarytool/stapler** for macOS signing (port of today's flow; reference `.resources/hermes/scripts/notarize.mjs`).

## Impact Analysis

| Component | Impact Type | Description and Risk | Required Action |
| --- | --- | --- | --- |
| `internal/update` | modified | Desktop-app installs become self-applicable; provenance writer; intent staging; multi-target state. Medium risk (update path) | Extend + full unit matrix |
| `internal/cli/update.go` | modified | Multi-target record, `blocked`, aggregate precedence. Low risk | Rewrite around `MultiState`; golden `_dx.md` transcripts |
| `internal/cli/app.go` | modified | `update` subcommand deleted; other verbs untouched. Low risk | Delete + docs/tests sweep |
| Settings surface (core/httpapi/udsapi/spec) | modified | Extended GET + new POST apply. Low risk | Contract co-ship (`make codegen`) |
| `web/` settings + os systems | modified | Two-track section, apply action, menubar indicator. Low risk | Per `_uiux.md` |
| `desktop/src-tauri` (all of it) | deleted | Replaced by `desktop/` Electron app. High risk concentrated in P3 parity | Full rewrite from blueprint |
| `internal/desktoprelease` | modified | Channel-config/feed generation deleted; inventory/version policy rewritten for per-arch set | Shrink + new inventory |
| Release CI (desktop lanes) | replaced | tauri-action → electron-builder per-arch + merge + GH publish; smoke adapted | New workflow lanes |
| Docs (`packages/site`) | modified | Desktop pages truth pass; `cli/app/update.mdx` deleted; `cli/update.mdx` multi-target (+`--cancel`); Chromium-first note; **new release-operator runbook `operations/desktop-release.mdx`** (publish/repair, known-good selection, CAS-conflict recovery, audit inspection — the authority's named co-ship owner, N-012) | Per US-028 |
| QA (`docs/qa`) | modified | All `APP-*` reset to `untested`; `REL-beta-self-update` reset; charters lose `tauri-driver` | QA pair tasks |

**Delete targets (no fallback, no compat shim, no placeholder):** `desktop/src-tauri/**`; Rust toolchain in CI (`desktop-build/test/lint` cargo lanes); `tauri-action` usage and `TAURI_*`/minisign secrets; `internal/desktoprelease/channel_config.go` + `cmd/compozy-desktop-release` subcommands `channel-config`/`assert-comparator`; `scripts/{normalize-desktop-artifacts.sh,assert-desktop-signing-material.sh,verify-desktop-release-build-contract.sh,generate-desktop-update-key.sh,verify-desktop-signature.sh,publish-desktop-release.sh,repair-desktop-feed.sh}`; `.github/workflows/desktop-feed-repair.yml`; `releases.compozy.com` feed objects + bucket/DNS (ops follow-through); `compozy app update` verb, tests, `packages/site/content/docs/cli/app/update.mdx` + meta rows; the shell's `update.check`/`update.apply` control methods; the `runtime.json` sibling-feed concept (its compatibility gate **survives** in the new form — signed `compat.json` asset + `/api/status` `min_app_version` handshake — only the old feed format dies, per B-004); the pre-review `app-update-intent.json` design (superseded by the Update Operation record before implementation); WebKitGTK troubleshooting content in `operations/desktop-app.mdx`. The deleted publish/repair scripts and `desktop-feed-repair.yml` are replaced by `compozy-desktop-release publish|repair` (Publish and repair authority above). No alias, dual field, or frozen artifact survives for any of these.

## Extensibility Integration Plan

No extension surface changes: extension manifests, hooks, skills/capabilities, tools/resources, registries, bridge SDKs, and MCP sidecars were checked — none references the desktop shell, and this program adds no extension points to it (Part I constraint). Two adjacent updates are required and carried as tasks: the **official Compozy skill** (`skills/compozy/`) must reflect the deleted verb, the multi-target `compozy update`, and the new apply endpoint; and the generated CLI/OpenAPI docs regenerate. Native `compozy__*` tools: checked — no tool exposes update or app-shell operations today; none is added (CLI/HTTP are the agent paths).

## Agent Manageability Plan

Consistent with `_dx.md` (the authority for shapes):

| Operation | Surface | Structured output | Deterministic failures |
| --- | --- | --- | --- |
| Inspect app | `compozy app status -o json` | app-state schema v1 | frozen taxonomy |
| Open/navigate | `compozy app open [path]` | `{ok:true}` / error envelope | `invalid_target_path`, `app_not_installed`, `app_not_running` |
| Update everything | `compozy update [-o json]` / `--check` | multi-target record, aggregate precedence | `blocked` (holder named), `failed` (record first, exit 1) |
| Update via API | `GET/POST /api/settings/update[/apply]` (HTTP+UDS) | extended status + apply result | `blocked`/`failed` bodies; 400 unknown target |
| Repair boot | `compozy app retry` | `{ok:true}` | `app_not_running` |
| Evidence | `compozy app diagnose [--bundle --yes]` | diagnostic report / bundle path | `diagnostic_consent_required` |
| Cancel dormant update | `compozy update --cancel` / `POST /api/settings/update/cancel` | cancel result with archived outcome | live-lease decline naming the holder |
| Channel operations | `compozy-desktop-release publish\|repair -o json` (release operator) | `{operation, channel_head_before/after, verified_inventory, audit_ref, outcome}` | `inventory_incomplete`, `channel_lock_held`, `verification_failed` |

## Config Lifecycle

No `config.toml` changes: `[app] update_check` + `update_check_interval` keep their keys, defaults (`true`, `6h`), bounds (15m–168h), merge behavior, validation errors, examples, and docs (`configuration/config-toml.mdx`) — verified against `internal/config/app.go`, `defaults.go`, `merge_app.go`. What changes is the **consumer set, reduced to one** (B-015): the daemon is the sole consumer of both keys, scheduling background checks for both tracks. The shell reads no update configuration — it reacts only to the durable operation record and reports execution state, so no second scheduler, no second clamp implementation, and no mirrored TypeScript validation exist. Existing config tests stay valid; one consumer-side test asserts the daemon cadence honors the keys.

## Compozy Impact Audit

- **Native tools**: no impact — checked the `compozy__*` registry and toolsets: no tool exposes update, app-shell, or release operations today and none is added; the agent paths are the CLI verbs and HTTP/UDS routes in the Agent Manageability Plan.
- **Extensibility and hooks**: no extension-surface change — extension manifests, hooks, skills/capabilities, tools/resources, registries, bridge SDKs, and MCP sidecars checked; none references the desktop shell and no extension point is added. The official Compozy skill (`skills/compozy/`) updates for the deleted verb, the single update command, and the new apply route — carried as an explicit task.
- **Workspace data isolation**: every datum this program creates or moves is **host-global** by design — the Update Operation record, update history, provenance, `app.json`, `update.lock`, update caches, and the settings update projection all describe the host install shared by every workspace. `workspace_id` is intentionally absent: `GET/POST /api/settings/update*` live in the global settings scope (no workspace parameter), the web surface renders under global Settings, no SSE/event stream carries update state, and no cache key is workspace-scoped. Checked paths: CLI (`compozy update`, `compozy app *` — home-scoped), HTTP/UDS settings routes, web settings query keys, `~/.compozy/cache`. Evidence: `_tests.md` UT-082 asserts the update surface exposes no workspace-scoped parameter, key, or route and that projections are identical across workspaces on one home.
- **Official Compozy skill**: `skills/compozy/` requires updates (deleted verb, `compozy update` multi-target record, apply endpoint, new statuses); checked that no other bundled skill references `compozy app update`.

## Testing Approach

Strategy only (cases live in `_tests.md`):

- **Go unit** (`make test`): `internal/update` matrix extends (desktop-app method apply, provenance rewrite, intent write/consume semantics, aggregate precedence, lock contention → `blocked`); CLI record/golden-output tests against the exact `_dx.md` transcripts; settings controller unit with a fake updater.
- **Web unit** (`bunx turbo run test --filter=./web`): section state machine per `_uiux.md` S1 states; indicator visibility rules; canonical suites extended (`-general.test.tsx` family), no parallel suites.
- **Integration** (`make test-integration`): daemon apply-runtime end-to-end against a local mock release server (reusing `internal/update` test fixtures) including restart coordination and rollback.
- **E2E**: new `make test-e2e-desktop` lane — Playwright `_electron` drives the real packaged shell against a real daemon (reference `.resources/hermes/.github/workflows/e2e-desktop.yml`), with a mock GitHub feed for update journeys; existing daemon-served browser suite extends for the settings/menubar surfaces; release smoke (`smoke-desktop-release-artifact.sh` successor) keeps the isolated-home boot gate.
- **Release gate** (manual, recorded in QA): full `APP-*` walk on macOS + Linux and one real beta N→N+1 auto-update cycle.

## Development Sequencing

### Build Order

- **P1 — Runtime-side unification** (Tauri still ships, untouched): Update Operation journal + companion lock + transition API (CAS/fencing) + executor lease + plan + resume/cancel + coordinator (ADR-009) with boot/launch/periodic recovery; extend `internal/update` (desktop-app apply, provenance, verified identity); rewrite `compozy update` (+`--cancel`); settings GET/POST/cancel with the three-projection contract + OpenAPI/TS; Go bootstrap/supervision entry point (`compozy daemon bootstrap -o jsonl` typed phases, bundle-aware); `compat.json` production + plan-shaping gate; daemon-served CSP with the exact production policy. Gate: `make gate` + update/REL unit-integration suites green (incl. writer-race IT-018 and recovery IT-019); `REL-beta-self-update` re-walked.
- **P2 — Web update surface**: S1/S2 per `_uiux.md` with design pass, rendering the operation lifecycle (phases, percent, holder); app-track apply affordance gated on an operation-capable shell generation (truthful UI while Tauri is live). Gate: web suites + browser E2E for the section.
- **P3 — Electron shell**: `desktop/` app at parity — bundled-runtime bootstrap launch + typed-progress rendering, control subset with the hardened socket + capability token, deep links, single instance, window-state, zoom, nav guards, security invariants 13–15 + boot-window CSP, app updater bound to the operation record (no shell config reader); `make test-e2e-desktop` lane. Gate: all `APP-*` scenarios walkable locally green, e2e lane in CI.
- **P4 — Pipeline + cutover**: per-arch builds, signing/notarization, manifest merge, `compozy-desktop-release publish|repair` authority, lockstep GH publish, smoke, repair rehearsal; then the single cutover change set executing every delete target; docs truth pass; QA full walk; real N→N+1 cycle. Gate: Part I release gate + `make gate-full`.

Safe-cleanup vs behavior-change separation: every deletion lands in P4's cutover change set, after P3 proves parity — never interleaved with behavior edits.

### Technical Dependencies

Apple signing/notarization credentials (already provisioned); GitHub Releases as the feed (public repo — already true); two mac CI builders (arm64 + x64); no external blockers otherwise. Windows explicitly parked (Azure account).

## Monitoring and Observability

- Canonical operation events (N-002/N-011), one per lifecycle edge, correlated by `operation_id`: `update.operation.acquired`, `update.operation.blocked`, `update.operation.waiting` (entered waiting-for-app), `update.operation.resumed`, `update.operation.canceled`, `update.operation.cancel_declined`, `update.lease.acquired`, `update.lease.renewed`, `update.lease.expired`, `update.check.completed`, `update.download.progress` (bounded cadence), `update.verify.completed`, `update.swap.completed`, `update.restart.completed`, `update.health.completed`, `update.finalized`, `update.rolled_back`, `update.app.staged`, `update.app.applying`, `update.app.installer_handoff`, `update.app.verified`, `update.app.failed`, `update.operation.recovered`. Required fields on every event: `operation_id`, `revision`, `target`, `install_method`, `from_version`, `to_version`, `actor` (`cli|daemon|web|shell`), `executor_generation` where a lease exists, `outcome`. Never secrets, never raw tokens.
- A lifecycle **coverage-matrix test** (UT-080) fails when any journaled phase transition lacks its canonical event — the same discipline `internal/CLAUDE.md` requires for domain operations.
- **Dispatch ownership (N-007)**: events are emitted at the journal-transition call site — the transition API is the sole emit owner. Nothing tails `update-history.jsonl` (or any log/event artifact) to synthesize events; the archive is evidence, never a dispatch source.
- The `update-history.jsonl` archive is the durable audit trail; the live operation record is the queryable in-flight truth (served by GET).
- Shell logs stay at `~/.compozy/logs/desktop.log` (rotating), with per-launch run id and renderer-crash records; `compozy app diagnose --bundle` continues to collect them.
- Failures surface in `compozy app status`'s update block, the web section's last-error row, and the operation view — one truth, three renderings.

## Technical Considerations

### Key Decisions

- **Direct `loadURL(daemonOrigin)`** over a privileged-scheme proxy: parity with today, zero new moving parts, CSP belongs daemon-side later; revisit only if port-change origin instability materializes (t3code's proxy is the documented fallback).
- **Operation record over daemon→shell IPC** for app applies (ADR-009): one durable mechanism covers running (record watch) and closed (next-launch recovery) shells; ownership survives every process exit; no inversion of the control-channel client/server roles.
- **Static electron-builder config + minimal generator** (channel/signing inputs only): reproducible `bunx electron-builder` locally, unlike the fully generated configs in the references.
- **No preload in the product window**: the SPA stays shell-agnostic; only the boot window gets a minimal, shape-validated bridge (port of `BOOT_BRIDGE_SCRIPT`).
- **Compatibility is two-sided and survives the feed cut** (B-004): shell-side `MINIMUM_RUNTIME` (build-config sourced) + runtime-side `min_app_version` on `/api/status` + the signed `compat.json` gate in the coordinator. Only the `runtime.json` transport dies.
- **Electron pinned exact; quarterly bump routine** budgeted as maintenance (three supported majors, ~24 weeks each).

### Known Risks

- Mac update-zip symlink corruption breaks Squirrel signature validation — pipeline repack guard + smoke (synara evidence).
- electron-updater mac download stalls (no idle timeout upstream) — accept initially; adopt the resumable-downloader pattern only if observed (`.resources/synara/apps/desktop/src/resumableUpdateDownload.ts`).
- First-generation chicken-egg for the N→N+1 gate — plan two consecutive beta releases in P4.
- deb installs get no app-track auto-update (updater supports AppImage only on Linux) — surfaced truthfully as a recommendation, documented.

## Safety Invariants

1. **Acquisition is creation.** Every mutation of the runtime install or the app bundle begins by atomically creating `~/.compozy/update-operation.json` (`O_EXCL`); observing lock or record state is never sufficient. A failed create returns `blocked` with the recorded holder — identically on CLI, HTTP/UDS, and web (B-003).
2. One live operation per home; both tracks live inside it, ordering fixed runtime-first; no second operation can interleave targets.
3. Verification — sigstore bundle → sha256 catalog → size policy → compatibility gate (`compat.json`) — completes before any byte of the current install moves; failure journals `failed` and leaves the install untouched.
4. **The executor's lifetime is never the daemon's.** The coordinator is a separate process; swap → restart → health-check → finalize/restore run under one owner across daemon generations. The daemon acquires, spawns detached, and serves the journal — it never swaps itself in-process (B-002; the L-001 lesson generalized: detaching from a request is not detaching from a process).
5. Every phase transition is journaled atomically (temp+rename under flock) before its action counts as done; recovery — daemon boot sweep and shell launch sweep — is a pure function of the journaled phase.
6. Any post-swap failure (restart not `ready`, health-check failed) triggers `Restore` from the journaled `backup_path` and records `rolled-back` with the restored version.
7. The app consumer transitions `staged → applying` inside the record **before** invoking the installer, installs only the recorded asset digest, and writes `verified` only after a post-restart version check; consumption alone never reports `updated` (B-006).
8. App-track and runtime-track mutations never overlap: the app phase cannot enter `applying` while the runtime phase is non-terminal — enforced by the record, not by lock observation.
9. Provenance rewrite happens inside the coordinator's swap step (same flock scope), so install-method detection never observes a half-updated state.
10. Version monotonicity holds at check (never offer ≤ current) and at apply (refuse downgrade) on every path; the compatibility gate re-checks under the operation immediately before mutation.
11. Publication ordering: payload assets upload and verify before the channel manifest flips, within one release and one CI concurrency group.
12. After cutover, the shell writes `~/.compozy/bin` only during first-run bootstrap into an empty location; every subsequent mutation belongs to the coordinator.

**Security invariants (shell + control channel — B-007):**

13. Every BrowserWindow ships `contextIsolation: true`, `sandbox: true`, `nodeIntegration: false`, and no `webviewTag`; the product window has no preload; the boot window's preload validates every message shape in both directions.
14. Permission request **and** check handlers are installed default-deny on every session; packaged builds disable devtools and cannot open a remote-debugging port under any environment combination.
15. Per-window navigation policy: `will-navigate` allows same-origin only, `setWindowOpenHandler` always denies, and only URLs that parse to http/https reach the OS browser; no other scheme is ever executed.
16. The daemon serves the SPA with an **exact production CSP** (ships in P1), identical in browser and app — `default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; font-src 'self'; connect-src 'self'; worker-src 'self' blob:; frame-src 'none'; frame-ancestors 'none'; object-src 'none'; base-uri 'self'; form-action 'self'` — no `unsafe-eval` anywhere, and `frame-ancestors 'none'` so no external page can frame the localhost UI (which now carries a mutation action — B-026). The packaged boot window ships its own stricter **meta-delivered** policy (`default-src 'none'; script-src 'self'; style-src 'self'; img-src 'self'; base-uri 'none'; form-action 'none'` — `frame-ancestors` does not apply via meta; the boot window additionally refuses any embedding by never being navigable). Dev-server relaxations are impossible in packaged builds. Tests assert **negative cases on both policies** (inline script blocked, unlisted connect origin blocked, framing refused), never mere header presence (B-019/B-026).
17. `app.sock` lives in a canonicalized, non-symlink parent directory owned by the user at `0700`; the socket is `0600`; every request is authenticated by the per-session capability token (`~/.compozy/app.token`, `0600`, timing-safe comparison, rotated per shell start, deleted on exit, never logged) — the implementable same-uid boundary chosen over unavailable peer-credential APIs (B-016); stale sockets are removed only after a failed connect probe plus ownership verification.
18. Control connections are bounded — max concurrent connections, 64 KiB message cap, one in-flight request per connection, write backpressure — and no control or diagnostic response ever contains secrets.

## Assumptions and Defaults

- **Lockstep versioning**: app and runtime ship from one tag; the app's latest version is read from the same GitHub Release as the runtime's.
- **Channel**: `beta` only publishes (policy unchanged); `stable` reserved.
- **Electron version**: latest stable at implementation time, pinned exact; bumps are routine maintenance, not spec changes.
- **Linux app-track auto-update**: AppImage only; deb reports the manual path (parity with updater capabilities).
- **Boot window**: packaged local page (port of `pages/boot.*`) with the reduced state set; strings frozen per `_dx.md`.
- **Web polling**: existing settings query cadence at rest; 2s interval while an operation is live; no SSE addition for update state in this program.
- **Operation record**: `~/.compozy/update-operation.json` (schema_version 1) + `~/.compozy/logs/update-history.jsonl` archive; the pre-review intent-file design is superseded (ADR-009).
- **App-state schema**: takes its one designed step to `schema_version: 2` (value-set extensions + `operation_id`/`phase`/`percent`), shipped lockstep across shell and CLI; mismatches fail with the existing `app_state_schema_unknown` error (B-023 hard cut).
- **Channel mechanics**: app-updater feed = generic provider over raw manifests on the `channel-beta` git branch; the channel switch/repair is a ref CAS; audit = commit history keyed by operation id (B-024).
- **Bundle trust root**: macOS = signed/notarized package covers the bundled runtime; all platforms = shell pre-spawn digest check against a build-embedded manifest; Linux package authenticity roots in download checksums (B-027).
- **E2E lane name**: `make test-e2e-desktop`.
- **CSP for the daemon-served UI**: in scope, ships in P1 with the exact policy in security invariant 16; boot window carries its own stricter policy.
- **Bundled runtime**: the app package embeds the lockstep per-arch `compozy` binary (installer grows by roughly the runtime size); first run installs from the bundle offline — no first-run network dependency.
- **Control-channel auth**: per-session capability token at `~/.compozy/app.token` (`0600`), timing-safe, rotated per shell start; the `token` request field ships lockstep within `schema_version: 1`.
- **`compat.json` fields**: exactly `{runtime_version, min_app_version}` — `min_app_version` is the sole compatibility authority (N-008).
- **Ops follow-through**: `releases.compozy.com` bucket/DNS teardown executes with the cutover task, after the last Tauri artifact is deleted.

## Architecture Decision Records

- [ADR-001: Replace the Tauri desktop shell with an Electron shell](adrs/adr-001.md) — engine parity is the product problem; Chromium everywhere.
- [ADR-002: Relocate runtime management out of the shell before the swap](adrs/adr-002.md) — thin-shell sequencing de-risks the updater.
- [ADR-003: No migration bridge; hard-cut the old feed; preserve identity](adrs/adr-003.md) — zero-legacy cutover for a ~zero installed base.
- [ADR-004: One runtime self-update mechanism, two consumers](adrs/adr-004.md) — app and CLI converge on `internal/update`.
- [ADR-005: One update command; `compozy app update` deleted](adrs/adr-005.md) — `compozy update` covers runtime and app.
- [ADR-006: Update offers live in the web UI; overlay non-interactive](adrs/adr-006.md) — daemon truth, settings + menubar, zero SPA shell-awareness.
- [ADR-007: App updates ship from GitHub Releases; releases.compozy.com retired](adrs/adr-007.md) — one release train, one trust chain.
- [ADR-008: macOS artifacts per-architecture](adrs/adr-008.md) — smaller downloads, reference-proven pattern.
- [ADR-009: Durable per-home Update Operation + detached coordinator](adrs/adr-009.md) — ownership survives every process exit; every surface converges on one primitive.
