# User Stories: Electron Desktop Shell Migration

Canonical behavior catalog for the Tauri→Electron desktop shell migration. Companion to `_spec.md`; consumed by `_spec.md` Part II (component mapping), `_uiux.md` (surface states, if UI-bearing), and `_tests.md` (coverage matrix).

Parity with three decided exceptions (ADR-005/ADR-006) is the migration contract: most stories describe existing behavior that must survive the shell swap unchanged. Stories marked **(new)** describe the genuinely new capabilities: the single `compozy update` command with headless self-update, the web-visible update surface, engine parity, and the cutover itself.

## Personas

- **Desktop Operator** — a developer on macOS or Linux who installs the CompozyOS desktop app and uses it as their daily interactive surface.
- **Headless Operator** — a human or AI agent that inspects, operates, and repairs the app and runtime through `compozy app` verbs with structured output, possibly with no GUI session at all (SSH, automation, agent-driven repair).
- **Release Operator** — the maintainer who publishes desktop releases, owns the channel feeds, and executes the cutover.

## Story Index

| ID     | Feature Area        | Persona          | Story                                                 |
| ------ | ------------------- | ---------------- | ----------------------------------------------------- |
| US-001 | Install & Boot      | Desktop Operator | Fresh install and first run provisions the runtime    |
| US-002 | Install & Boot      | Desktop Operator | New installer replaces an old-generation install      |
| US-003 | Install & Boot      | Desktop Operator | App attaches to an already-running runtime            |
| US-004 | Install & Boot      | Desktop Operator | App starts an installed, stopped runtime              |
| US-005 | Install & Boot      | Desktop Operator | Boot failure explains itself and offers retry         |
| US-006 | Install & Boot      | Desktop Operator | Quitting the app leaves the runtime running           |
| US-007 | Window & Rendering  | Desktop Operator | Second launch focuses the existing window             |
| US-008 | Window & Rendering  | Desktop Operator | Window geometry survives restart                      |
| US-009 | Window & Rendering  | Desktop Operator | Page zoom controls with persistence                   |
| US-010 | Window & Rendering  | Desktop Operator | External links open in the system browser             |
| US-011 | Window & Rendering  | Desktop Operator | Browser and app render identically **(new)**          |
| US-012 | Window & Rendering  | Desktop Operator | UI crash recovers without losing the runtime          |
| US-013 | Deep Links          | Desktop Operator | Deep link navigates the running app                   |
| US-014 | Deep Links          | Desktop Operator | Deep link cold-starts the app                         |
| US-015 | Updates             | Desktop Operator | App updates itself on the beta channel                |
| US-016 | Updates             | Desktop Operator | Failed app update recovers safely                     |
| US-017 | Updates             | Desktop Operator | Runtime updates with progress and rollback            |
| US-018 | Updates             | Headless Operator | Runtime self-updates with no app installed **(new)** |
| US-019 | Updates             | Desktop Operator | Update behavior obeys configuration                   |
| US-020 | CLI Manageability   | Headless Operator | Inspect app state with `compozy app status`          |
| US-021 | CLI Manageability   | Headless Operator | Open the app at a destination with `compozy app open` |
| US-022 | CLI Manageability   | Headless Operator | Drive updates with the single `compozy update`       |
| US-023 | CLI Manageability   | Headless Operator | Retry a failed boot with `compozy app retry`         |
| US-024 | CLI Manageability   | Headless Operator | Diagnose and export evidence with `compozy app diagnose` |
| US-025 | Release Operations  | Release Operator | Publish a beta release through the new pipeline       |
| US-026 | Release Operations  | Release Operator | Repair or roll back a bad feed                        |
| US-027 | Release Operations  | Release Operator | Execute the cutover **(new)**                         |
| US-028 | Docs                | Desktop Operator | Docs describe the shipped shell truthfully **(new)**  |
| US-029 | Updates             | Desktop Operator | Update state visible and appliable in the web UI **(new)** |

## Install & Boot

### US-001: Fresh install and first run provisions the runtime

**As a** Desktop Operator, **I want** a fresh install to boot the app, provision its own runtime, and land me in the UI, **so that** installing the app is the complete setup.

Acceptance criteria:

- AC-1: Given a machine with no runtime installed, when the operator opens the freshly installed app, then a boot window appears immediately and shows real progress phases (resolve → provision → start → ready) until the main window opens on the product UI.
- AC-2: Given a macOS machine, when the operator installs from the disk image and opens the app, then first run completes without any additional tooling.
- AC-3: Given a Linux machine, when the operator installs from either supported package format and launches, then first run completes identically.
- AC-4: Given first run completed, when the operator inspects app state via CLI, then the record shows the app-owned runtime, its version, and a healthy status.
- AC-5: Given `COMPOZY_HOME` is overridden in the environment, when the app boots, then all provisioning and state land under the overridden home.
- AC-6: Given no network connectivity, when the operator opens the freshly installed app, then first run still completes — the runtime installs from the app package itself.

Edge cases:

- EC-1: Bundled runtime payload missing or corrupt (damaged download/disk) → provision refuses with a clear reinstall-the-app explanation; nothing partially installed is ever left executable.
- EC-2: Bundled runtime binary fails integrity verification → provision refuses to install it, surfaces the failure, and leaves no runtime in place.
- EC-3: Disk exhausted during install-from-bundle → failure phase names the cause; retry after freeing space succeeds.
- EC-4: A runtime newer than the app's supported range is already present → the app refuses attach with a version-mismatch explanation instead of undefined behavior.
- EC-5: First run launched twice in quick succession (double-click) → exactly one app instance and one provision flow run.

### US-002: New installer replaces an old-generation install

**As a** Desktop Operator with the old-generation app installed, **I want** the new installer to replace it in place, **so that** switching generations is a normal reinstall, not a migration project.

Acceptance criteria:

- AC-1: Given the old app is installed on macOS, when the operator installs the new disk image, then the new app replaces the old one under the same name and identity, and launching it attaches to the same home, workspaces, and runtime state.
- AC-2: Given the old app is installed via the Linux package, when the operator installs the new package, then it upgrades in place under the same package identity.
- AC-3: Given the replacement completed, when the operator runs `compozy app status`, then the reported app is the new generation with the same install identity.

Edge cases:

- EC-1: Old portable Linux binary (single-file format) left on disk next to the new one → both exist as files; the announcement instructs deleting the old file; `compozy app status` reports the newly registered install.
- EC-2: Old app still running while the new one is installed → the new app's first launch proceeds normally after the old one quits; no shared-state corruption.
- EC-3: Replacement performed while the runtime is mid-task → runtime is untouched by app replacement; sessions continue.

### US-003: App attaches to an already-running runtime

**As a** Desktop Operator whose runtime is already running (started by CLI or a previous session), **I want** the app to attach to it instantly, **so that** daily opens are fast and never duplicate runtimes.

Acceptance criteria:

- AC-1: Given a healthy runtime is running, when the app launches, then it attaches without starting a second runtime and the main window opens against the running instance.
- AC-2: Given attach succeeded, when the operator checks running processes, then exactly one runtime process exists.

Edge cases:

- EC-1: Running runtime is older than the app's minimum supported version → app refuses attach and explains the version gap with the repair path.
- EC-2: Runtime is listening but unhealthy (status endpoint failing) → app treats it as not-attachable and follows the start/repair path with diagnostics, never a hang.
- EC-3: Stale runtime record (process died, record left behind) → app detects the stale record, cleans its view, and proceeds to start.

### US-004: App starts an installed, stopped runtime

**As a** Desktop Operator with a runtime installed but not running, **I want** the app to start it on launch, **so that** the app is always a one-click path into a working system.

Acceptance criteria:

- AC-1: Given an installed, stopped runtime, when the app launches, then the boot window shows the start phase and the main window opens once the runtime reports ready.
- AC-2: Given the runtime was started by the app, when the app quits, then the runtime keeps running (see US-006).

Edge cases:

- EC-1: Runtime binary present but not executable/corrupt → boot fails into diagnostics with the exact phase and a repair suggestion.
- EC-2: Start succeeds but readiness never arrives within the supervision policy → bounded retries with backoff, then a failure dialog offering retry, open logs, and quit — never an infinite spinner.
- EC-3: Another process occupies the runtime's port → failure names the conflict rather than silently starting on undefined state.

### US-005: Boot failure explains itself and offers retry

**As a** Desktop Operator, **I want** every boot failure to say which phase failed, why, and what to do, **so that** I can repair without reading source code.

Acceptance criteria:

- AC-1: Given any boot phase fails, when the failure surfaces, then the boot window shows the failed phase, a human-readable cause, and actions: retry, open logs, quit.
- AC-2: Given a previous session crashed, when the app next boots, then the crash is recorded and visible in diagnostics output.
- AC-3: Given the operator chooses retry after fixing the cause, then boot proceeds from a clean state and succeeds.

Edge cases:

- EC-1: Retry pressed while the previous attempt is still winding down → attempts never overlap; the second request queues or is refused with feedback.
- EC-2: Repeated failures of the same cause → retries are bounded with backoff; the app never hot-loops provisioning or start attempts.
- EC-3: Retry issued when the app is already healthy → no-op with a clear "already running" result and no state damage (regression guard: healthy retry must never corrupt recorded state).

### US-006: Quitting the app leaves the runtime running

**As a** Desktop Operator, **I want** quitting the app to close only the window layer, **so that** CLI sessions, agents, and background work keep running.

Acceptance criteria:

- AC-1: Given the app started or attached to the runtime, when the operator quits the app, then the runtime process keeps running and remains reachable by CLI and browser.
- AC-2: Given the app is quit, when the operator relaunches it, then it attaches to the still-running runtime (US-003).

Edge cases:

- EC-1: Quit requested while a runtime update is being applied → the update completes or rolls back safely before the app exits; never a half-swapped runtime.
- EC-2: Quit during provision (first run incomplete) → provision aborts cleanly; next launch resumes or restarts provision without corruption.
- EC-3: OS shutdown/logout kills the app → same guarantee as manual quit: runtime state stays consistent.

## Window & Rendering

### US-007: Second launch focuses the existing window

**As a** Desktop Operator, **I want** launching the app again to focus the window I already have, **so that** I never end up with duplicate app instances.

Acceptance criteria:

- AC-1: Given the app is running, when the operator launches it again (dock, launcher, CLI), then the existing window is focused and un-minimized; no second instance starts.
- AC-2: Given the second launch carried a deep link, then the existing window both focuses and navigates to the link target.

Edge cases:

- EC-1: Rapid repeated launches → still exactly one instance; no focus flicker loop.
- EC-2: First instance is unresponsive → the second launch does not spawn a duplicate; the unresponsive instance remains the single owner until the OS or user kills it.

### US-008: Window geometry survives restart

**As a** Desktop Operator, **I want** the window to reopen where I left it, **so that** the app feels persistent.

Acceptance criteria:

- AC-1: Given the operator moved/resized the window, when the app restarts, then position, size, and maximized state are restored.
- AC-2: Given the boot window, then its geometry is fixed and never recorded into saved state.

Edge cases:

- EC-1: Saved geometry belongs to a display that is now disconnected → the window is clamped/centered onto a connected display, never restored off-screen.
- EC-2: Saved state file is corrupt or hand-edited into garbage → defaults apply silently; the file is rewritten healthy on next close.
- EC-3: Display scale (DPI) changed between sessions → the window still lands fully visible.

### US-009: Page zoom controls with persistence

**As a** Desktop Operator, **I want** zoom in/out/reset with my level remembered, **so that** the UI matches my eyesight and display.

Acceptance criteria:

- AC-1: Given the app is focused, when the operator uses the standard zoom shortcuts or menu items, then the UI zooms in steps within bounded limits and shows the standard behavior for reset.
- AC-2: Given a zoom level was set, when the app restarts, then the level is restored.

Edge cases:

- EC-1: Zoom at the boundary → further zoom in the same direction is a no-op, no error.
- EC-2: UI reload (crash recovery, update) → zoom level re-applies; it never silently resets to default.

### US-010: External links open in the system browser

**As a** Desktop Operator, **I want** links leaving the product to open in my default browser, **so that** the app never becomes an accidental web browser.

Acceptance criteria:

- AC-1: Given any link to an origin other than the product UI, when clicked (or opened via window.open), then it opens in the OS default browser and the app window stays on the product.
- AC-2: Given a link within the product UI origin, then it navigates in-app normally.

Edge cases:

- EC-1: Link with a non-web scheme (mail, custom app schemes, file) → refused or delegated per an allowlist; never executed by the app window.
- EC-2: Hostile page content attempts top-level navigation to an external origin → blocked in-app; pushed to the system browser only when it parses as a safe web link.

### US-011: Browser and app render identically (new)

**As a** Desktop Operator, **I want** the app to render exactly what the browser surface renders, **so that** there is one UI truth on every platform.

Acceptance criteria:

- AC-1: Given the same runtime and UI version, when a screen renders in the supported browser and in the app, then layout, effects (including glass/blur depth effects), and interactive behavior are identical.
- AC-2: Given a Linux machine (including past problem configurations), when the app renders translucency/blur-bearing surfaces, then they render correctly with no environment-variable workarounds.
- AC-3: Given the automated browser E2E suite passes a journey, then the same journey passes when driven against the installed app.

Edge cases:

- EC-1: GPU acceleration unavailable (VM, remote session) → the app falls back to software rendering and stays usable; no blank window.
- EC-2: Exotic display stack (Wayland vs X11) → rendering parity holds on both without per-user tuning.

### US-012: UI crash recovers without losing the runtime

**As a** Desktop Operator, **I want** a UI-layer crash to self-recover, **so that** a rendering fault never costs me running work.

Acceptance criteria:

- AC-1: Given the UI layer crashes, when recovery runs, then the window reloads automatically within a bounded budget and the operator's sessions are intact (runtime untouched).
- AC-2: Given the reload budget is exhausted, then a dialog offers reload, open logs, and quit — with the crash recorded for diagnostics.

Edge cases:

- EC-1: Crash loop (crash immediately after every reload) → bounded attempts within a time window, then the dialog; never an infinite reload loop.
- EC-2: Window unresponsive (hang, not crash) → detected and logged distinctly from a crash so diagnosis differs.

## Deep Links

### US-013: Deep link navigates the running app

**As a** Desktop Operator, **I want** `compozyos://` links to land the running app on the right screen, **so that** the CLI and future surfaces can hand me exact destinations.

Acceptance criteria:

- AC-1: Given the app runs, when a valid `compozyos://open<path>` link is opened, then the existing window focuses and navigates to that product path.
- AC-2: Given `compozy app open <path>` with the app running, then the same navigation happens via the control channel without spawning anything.

Edge cases:

- EC-1: Link carrying an external URL, host, traversal segments, backslashes, or scheme smuggling (`compozyos://open/http://evil.com` and kin) → rejected; the app lands on the default view, never on hostile content.
- EC-2: Link to a product path that does not exist → the UI's own not-found handling; never a crash or blank window.
- EC-3: Two links arrive in quick succession → last one wins; no interleaved half-navigation.

### US-014: Deep link cold-starts the app

**As a** Desktop Operator, **I want** a `compozyos://` link to launch the app when it is not running, **so that** links are reliable entry points, not "only if already open".

Acceptance criteria:

- AC-1: Given the app is installed but not running, when a valid link is opened, then the app launches, completes its boot ladder, and lands on the link target.
- AC-2: Given the link arrived before the UI was ready, then it is queued and applied exactly once when the UI signals ready — never dropped, never applied twice.

Edge cases:

- EC-1: Cold-start link with a hostile payload → app launches to the default view (same rejection rules as US-013.EC-1).
- EC-2: App not installed at all → the OS reports no handler; `compozy app open` reports the deterministic not-installed error instead of hanging.
- EC-3: Link opened during an in-flight boot triggered by a normal launch → single instance rule holds; the boot completes and then navigates.

## Updates

### US-015: App updates itself on the beta channel

**As a** Desktop Operator, **I want** the app to find, download, and apply its own updates on my command, **so that** staying current is one click.

Acceptance criteria:

- AC-1: Given a newer version exists on the channel, when the background check runs (default cadence) or the operator checks manually, then the update is surfaced with the target version in the product UI's Updates section and via the menubar indicator (US-029).
- AC-2: Given an update is surfaced, when the operator applies it (Updates section action or `compozy update`), then the app downloads with visible progress, verifies the artifact, restarts, and comes back on the new version with state intact.
- AC-3: Given no newer version exists, then check reports up-to-date; nothing downloads.
- AC-4: Downloads and installs happen only on explicit action — background checks never mutate the install.

Edge cases:

- EC-1: Feed unreachable (offline) → background check fails silently into logs and retries next interval; a manual check surfaces the error.
- EC-2: Downloaded artifact fails signature/integrity verification → update refused, current install untouched, deterministic error surfaced.
- EC-3: Version on the feed is lower than or equal to the installed version → never offered; downgrade is refused.
- EC-4: Linux package-format install without an updatable format → update check still reports availability, apply explains the manual path for that format.
- EC-5: Two apply requests race (UI + CLI) → exactly one proceeds; the second reports update-in-progress.
- EC-6: Update applied while a runtime update is in flight → mutually exclusive; the second track waits or is refused with a clear state.

### US-016: Failed app update recovers safely

**As a** Desktop Operator, **I want** a failed or interrupted update to leave me on a working version with an honest status, **so that** updating is never a gamble.

Acceptance criteria:

- AC-1: Given the app restarted after applying an update but still runs the old version (silent install failure), then the failure is detected on next boot, recorded, surfaced in update status, and automatic checks pause until a successful retry or new release.
- AC-2: Given an interrupted download, when the operator retries, then the download resumes or restarts cleanly and completes.
- AC-3: Given a failure was recorded, when the operator retries and it succeeds, then status returns to healthy and automatic checks resume.

Edge cases:

- EC-1: Quit-to-install fires but the app process never exits → detected within a bounded watchdog; the attempt is marked failed and the app returns to a usable state instead of limbo.
- EC-2: Machine crashes mid-install → next boot detects the incomplete attempt and reports version truth (whichever version actually runs).
- EC-3: Repeated failures of the same version → attempts are counted; status escalates to needs-attention rather than looping forever.

### US-017: Runtime updates with progress and rollback

**As a** Desktop Operator, **I want** runtime updates to run with progress and automatic rollback from whichever surface I use, **so that** the engine under my sessions stays current without risk.

Acceptance criteria:

- AC-1: Given a newer runtime exists for my platform, when the operator applies the runtime update (Updates section action or `compozy update`), then the system quiesces safely, swaps via the unified self-update mechanism, verifies health on the new version, and reports the new version.
- AC-2: Given the new runtime fails its post-swap health verification, then the previous runtime is restored automatically and the failure is reported with evidence.
- AC-3: Given sessions are active, when a runtime update is requested, then the operator is told what quiescing means before anything stops.

Edge cases:

- EC-1: Runtime update offered whose schema requirements exceed what the installed app supports → refused with a compatibility explanation, never a half-compatible install.
- EC-2: Update interrupted mid-swap (power loss) → next boot detects the journal and completes or rolls back; never a missing/corrupt runtime binary.
- EC-3: Runtime managed externally (operator-installed, not app-owned) → the app refuses to mutate an install it does not own and points to the headless path (US-018).
- EC-4: App update and runtime update requested together → ordered, never concurrent (same exclusivity as US-015.EC-6).

### US-018: Runtime self-updates with no app installed (new)

**As a** Headless Operator, **I want** the runtime to update itself via CLI on a machine with no desktop app, **so that** servers, CI, and agent-managed hosts stay current through the same mechanism the app uses.

Acceptance criteria:

- AC-1: Given a standalone runtime install and no desktop app, when the operator invokes the runtime self-update via CLI, then the same check/download/verify/swap/rollback behavior as US-017 executes and reports structured progress and a final version.
- AC-2: Given the app-driven path (US-017) and the headless path run against the same version pair, then their observable outcomes (installed version, rollback behavior, integrity refusals) are identical — one mechanism, two consumers.
- AC-3: Given the update completes, when the operator queries the runtime version, then it reports the new version and the update history records the transition.

Edge cases:

- EC-1: Headless update invoked while the desktop app is mid-runtime-update on the same home → single-flight: the second invocation observes the in-progress state and refuses with a deterministic error.
- EC-2: Headless update on an unsupported platform key → deterministic unsupported-platform error, nothing mutated.
- EC-3: Rollback path exercised headless (new version fails health check) → previous version restored, exit status and structured output say so explicitly.
- EC-4: Invoked with insufficient filesystem permissions on the install location → deterministic permission error, no partial swap.

### US-019: Update behavior obeys configuration

**As a** Desktop Operator, **I want** the existing update configuration keys to keep governing the new shell, **so that** my settings survive the migration untouched.

Acceptance criteria:

- AC-1: Given `[app] update_check = false`, then no background update checks run (app or runtime); manual checks still work.
- AC-2: Given `[app] update_check_interval` is set within its documented bounds, then background checks follow that cadence.
- AC-3: Given an out-of-bounds interval, then configuration validation fails with the same deterministic error path as today.

Edge cases:

- EC-1: Config changed while the app runs → next check honors the new value without restart, or the documented reload behavior applies unchanged.
- EC-2: Config file absent → documented defaults (checks on, default cadence) apply.

### US-029: Update state visible and appliable in the web UI (new)

**As a** Desktop Operator (or a browser-only operator on the same host), **I want** both update tracks visible and appliable from the product UI, **so that** staying current never requires a terminal or a shell overlay.

Acceptance criteria:

- AC-1: Given an update is available on either track, then the settings Updates section shows it (per track: versions, state, release link) and a discreet menubar indicator appears; activating the indicator lands on the section.
- AC-2: Given no update is available, then the menubar indicator is absent entirely.
- AC-3: Given the same daemon, then the section renders identically in the desktop app and in a plain browser — same states, same actions.
- AC-4: Given keyboard-only operation, then the indicator and the apply action are reachable and activatable.
- AC-5: Given the operator applies from the section, then per-track progress renders live (the staged phases), ending in the updated/staged/failed truth the daemon reports.

Edge cases:

- EC-1: Managed runtime install → the recommendation renders verbatim; the apply affordance is absent, not disabled.
- EC-2: App installed but closed, apply requested from a browser → runtime applies now; the app row shows `staged` with next-launch copy.
- EC-3: No desktop app installed on the host → the app row is absent; the section is single-track as today.
- EC-4: Update applying or failed → the menubar indicator stays hidden; progress and errors render only in the section.
- EC-5: Daemon restarting mid-apply (runtime swap) → the section reflects reconnect truthfully (checking/unavailable state), then resolves to the post-update truth.
- EC-6: Refresh failure while a snapshot exists → last-known state with an explicit refresh-failed marker and retry (today's behavior preserved).

## CLI Manageability

### US-020: Inspect app state with `compozy app status`

**As a** Headless Operator, **I want** `compozy app status` to report install, runtime, and update state as schema-valid structured output, **so that** agents and scripts can reason about the desktop surface without a GUI.

Acceptance criteria:

- AC-1: Given any app state (not installed, installed not running, running, updating, failed), when `compozy app status` runs, then the output validates against the published app-state schema and reflects reality.
- AC-2: Given the app is not installed, then status reports that deterministically without error exit.
- AC-3: Given the new-generation app is installed, then the reported identity/version reflect it under the announced v2 schema step (live-operation fields added); a v1 CLI meeting v2 state fails with the existing deterministic schema-unknown error, never a misread.

Edge cases:

- EC-1: Recorded state file is corrupt or from an unknown schema version → the deterministic schema-unknown error, never a crash or invented state.
- EC-2: State file absent but app installed → status reports installed-with-no-record coherently.
- EC-3: Status invoked concurrently with a state write → reader never sees a torn record.

### US-021: Open the app at a destination with `compozy app open`

**As a** Headless Operator, **I want** `compozy app open [path]` to focus-or-launch the app at a product path, **so that** terminal workflows can hand off to the GUI deterministically.

Acceptance criteria:

- AC-1: Given the app runs, then the command navigates it via the control channel and exits success.
- AC-2: Given the app is installed but not running, then the command cold-starts it via the deep-link path.
- AC-3: Given the app is not installed, then the deterministic not-installed error returns.

Edge cases:

- EC-1: Invalid path argument (relative, traversal, host-bearing, scheme-bearing) → deterministic invalid-target error before anything launches.
- EC-2: Control channel present but dead (stale socket) → deterministic not-running/unavailable error within the documented timeout, then cold-start fallback where defined today.

### US-022: Drive updates with the single `compozy update`

**As a** Headless Operator, **I want** one command that updates every applicable target on the host with structured per-target results, **so that** agents keep hosts current without knowing track internals.

Acceptance criteria:

- AC-1: Given `--check`, then the result reports current/available versions for the runtime and — when a desktop app is installed — the app, without mutating anything.
- AC-2: Given a default invocation, then every applicable target executes with the guarantees of US-015/US-017/US-018 and returns one structured record with per-target outcomes and the documented aggregate-status precedence.
- AC-3: Given the app is installed but not running, then the runtime updates now and the app update stages with an explicit `staged` status; the next app launch applies it.
- AC-4: Given `compozy app update` (the deleted verb), then the CLI reports an unknown command — no alias, no shim.
- AC-5: Given a dormant operation (an app update staged and never picked up, or a holder whose process died), when `compozy update --cancel` runs, then the operation cancels with an archived `canceled` outcome and the update channel frees; a live executor declines the cancel and names the holder.

Edge cases:

- EC-1: Invoked while another update is in flight (any surface) → deterministic `blocked` record naming the lease holder, non-zero exit.
- EC-5: Cancel racing a resume of the same dormant operation → exactly one wins; the record is never lost or duplicated.
- EC-2: App quits mid-app-apply → the record reports the interruption truthfully; the attempt resolves via US-016 recovery on next launch.
- EC-3: Host with no desktop app → the record contains no `app` object at all — absence, not an empty stub.
- EC-4: Managed runtime install (package manager) → recommendation only, nothing mutated, exit zero.

### US-023: Retry a failed boot with `compozy app retry`

**As a** Headless Operator, **I want** `compozy app retry` to re-drive a failed boot remotely, **so that** repair does not require touching the GUI.

Acceptance criteria:

- AC-1: Given the app sits in a failed boot state, when retry is invoked, then the boot ladder re-runs and the command reports the terminal outcome.
- AC-2: Given the app is healthy, then retry is a safe no-op reporting already-healthy — and never corrupts recorded state (regression guard).

Edge cases:

- EC-1: Retry when the app is not running → deterministic not-running error.
- EC-2: Retry racing an in-flight boot attempt → exactly one attempt runs; the command reports the winner's outcome.

### US-024: Diagnose and export evidence with `compozy app diagnose`

**As a** Headless Operator, **I want** structured diagnostics and a consent-gated evidence bundle, **so that** remote debugging has everything and leaks nothing.

Acceptance criteria:

- AC-1: Given any app state, when diagnose runs, then phases, previous crashes, and current status return as schema-valid structured output with secrets absent.
- AC-2: Given `--bundle` without consent, then the deterministic consent-required error returns and nothing is written.
- AC-3: Given `--bundle --yes` (and optional output path), then an evidence archive is written containing logs and state manifest, and its path is reported.

Edge cases:

- EC-1: Bundle output path unwritable → deterministic filesystem error, no partial archive left behind.
- EC-2: Diagnose while the app is mid-update → reports the in-flight update state truthfully instead of blocking.
- EC-3: Logs at rotation boundary during bundle capture → archive remains internally consistent.

## Release Operations

### US-025: Publish a beta release through the new pipeline

**As a** Release Operator, **I want** one pipeline that builds, signs, verifies, smoke-tests, and publishes app + runtime artifacts and feeds, **so that** releasing stays a routine, gated operation.

Acceptance criteria:

- AC-1: Given a release tag, when the pipeline runs, then it produces the exact expected artifact inventory for macOS and Linux, signed and (macOS) notarized, and refuses to publish on any missing signing material.
- AC-2: Given artifacts built, then a packaged-app smoke boot runs against an isolated home and gates publication.
- AC-3: Given publication, then payloads become available before channel manifests flip, so an updating client can never see a version whose artifacts are missing.
- AC-4: Given the published feed, then a client on version N (this generation) auto-updates to N+1 — proven once as part of the migration's release gate.

Edge cases:

- EC-1: Any signing/notarization secret missing → preflight fails the run before building; nothing partial publishes.
- EC-2: Version equal to or lower than the live channel version → publication refused (strict monotonicity).
- EC-3: Pipeline interrupted between payload upload and manifest flip → channel remains on the previous consistent state; rerun completes idempotently.

### US-026: Repair or roll back a bad feed

**As a** Release Operator, **I want** a deterministic feed-repair path, **so that** a bad publish is recoverable in minutes.

Acceptance criteria:

- AC-1: Given a bad manifest is live, when the repair procedure runs, then the channel returns to the last known-good state and clients resume updating from it.
- AC-2: Given a repair happened, then the recovery artifacts remain inspectable (audit trail).

Edge cases:

- EC-1: Repair invoked concurrently with a publish → one wins; the feed never serves an interleaved state.
- EC-2: Rollback target artifacts already garbage-collected → repair refuses with an explicit inventory error instead of publishing dead links.

### US-027: Execute the cutover (new)

**As a** Release Operator, **I want** a single cutover release that swaps generations completely, **so that** zero legacy remains.

Acceptance criteria:

- AC-1: Given the cutover release ships, then the new-generation artifacts and feeds are live, and the old-generation feed objects are deleted (not frozen).
- AC-2: Given the cutover lands in the repo, then every old-shell code path, build target, CI job, signing input, and helper script is deleted in the same change set (delete targets enumerated in the spec).
- AC-3: Given the cutover is announced, then the announcement states the manual re-download path and the old-install cleanup steps.

Edge cases:

- EC-1: An abandoned old install keeps polling the deleted feed → it logs failed checks on its own cadence; no server-side compatibility object is ever added for it.
- EC-2: Cutover discovered mid-failure (new feed bad) → feed repair (US-026) applies to the new generation; re-publishing old-generation artifacts is not a rollback path.

## Docs

### US-028: Docs describe the shipped shell truthfully (new)

**As a** Desktop Operator reading the docs, **I want** every desktop page to match the shipped app, **so that** troubleshooting and setup guidance actually work.

Acceptance criteria:

- AC-1: Given the migration ships, then the getting-started and operations desktop pages describe the new shell's behavior, and every old-engine workaround (environment-variable graphics fixes, old dev commands) is deleted.
- AC-2: Given the docs claim browser support, then a Chromium-first note exists: the product UI is developed and verified against the same engine the app ships; other browsers are best-effort.
- AC-3: Given the CLI reference pages, then the preserved `compozy app` verb pages match behavior byte-for-byte, the `compozy app update` page is deleted, and the `compozy update` page documents the multi-target command.

Edge cases:

- EC-1: A doc page referencing the old shell survives the sweep → treated as a defect of the cutover change set (US-027.AC-2), not a follow-up.
- EC-2: Config reference pages → `[app]` keys documented unchanged; any new key introduced by the migration is documented in the same change.
