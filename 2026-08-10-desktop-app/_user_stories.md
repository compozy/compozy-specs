# User Stories: CompozyOS Desktop App

Canonical behavior catalog for the CompozyOS desktop application. Companion to `_prd.md`; consumed by `_techspec.md` (component mapping) and `_tests.md` (coverage matrix).

## Personas

- **Desktop-first newcomer** — a developer whose first contact with CompozyOS is downloading the desktop app. Has no runtime installed, no home directory state, and expects "download → open → working product" with one coherent update experience afterward.
- **CLI power user** — an existing CompozyOS user who runs the runtime from the terminal, owns workspaces/sessions/local state, and today uses the product in browser tabs. Wants a native window without the app disturbing the runtime, its data, or CLI workflows.
- **Agent / automation operator** *(secondary)* — an agent or script that must launch, inspect, and reason about the desktop surface through structured command-line output, without a human at the screen.
- **Release operator** *(secondary)* — the team member who publishes desktop releases and must never be able to ship an update that installed clients would reject or that would strand them.

## Story Index

| ID     | Feature Area                | Persona                | Story                                                        |
| ------ | --------------------------- | ---------------------- | ------------------------------------------------------------ |
| US-001 | Install & first run         | Desktop-first newcomer | Install the app and get a working product with no prior setup |
| US-002 | Install & first run         | Desktop-first newcomer | Recover from a failed or interrupted first-run provisioning   |
| US-003 | Install & first run         | CLI power user         | App attaches to my already-running runtime and touches nothing |
| US-004 | Install & first run         | CLI power user         | App starts my installed-but-stopped runtime                   |
| US-005 | Runtime connection states   | CLI power user         | Incompatible app/runtime versions produce a guided state      |
| US-006 | Runtime connection states   | Any user               | Runtime unavailable states are honest and actionable          |
| US-007 | Runtime connection states   | Any user               | Runtime dying while the app is open degrades gracefully       |
| US-008 | Runtime connection states   | CLI power user         | Closing the app never kills my runtime or agent work          |
| US-009 | Window & native integration | Any user               | Launching again focuses the existing window                   |
| US-010 | Window & native integration | Any user               | CompozyOS links open inside the app                           |
| US-011 | Window & native integration | Any user               | External links open in my default browser                     |
| US-012 | Window & native integration | Any user               | Window geometry persists and recovers from invalid state      |
| US-013 | Window & native integration | Any user               | Launch is flash-free and never hangs on a loading screen      |
| US-014 | App auto-update             | Any user               | App updates arrive automatically and apply on my restart      |
| US-015 | App auto-update             | Any user               | A failed update never strands me silently                     |
| US-016 | Runtime update surfacing    | Desktop-first newcomer | App-provisioned runtime updates feel like one product update  |
| US-017 | Runtime update surfacing    | CLI power user         | Managed runtime installs get a recommendation, never a write  |
| US-018 | Channels                    | Any user               | I can see which release channel I am on                       |
| US-019 | Agent manageability         | Agent operator         | Launch and inspect the desktop surface from the command line  |
| US-020 | Browser coexistence         | CLI power user         | Browser and app show the same product with the same state     |
| US-021 | Release integrity           | Release operator       | An unpublishable release is blocked before users can see it   |

## Install & first run

### US-001: Install the app and get a working product with no prior setup

**As a** desktop-first newcomer, **I want** to install CompozyOS from a single download and reach a working UI, **so that** my first contact with the product does not require terminal setup.

Acceptance criteria:

- AC-1: Given a supported machine (macOS arm64/x64, Windows x64, Linux x64) with no runtime installed, when the user installs and opens the app, then a guided first-run flow provisions the runtime and the app lands in the full product UI without the user leaving the app.
- AC-2: Given the first-run flow is provisioning, when the user watches the screen, then each stage is visible with progress (no indefinite spinner), and the app window is branded CompozyOS throughout.
- AC-3: Given installation on macOS or Windows, when the OS evaluates the installer/app, then it is recognized as signed by a verified publisher (no unsigned-software wall).
- AC-4: Given provisioning completed once, when the user relaunches the app later, then no provisioning step reappears and the UI is reachable directly.

Edge cases:

- EC-1: Machine is offline at first run → provisioning states the network requirement and offers retry; the app does not present a broken half-installed state.
- EC-2: Provisioning target location is not writable (permissions) → the flow names the path and the reason and offers retry after the user fixes it; no silent partial install.
- EC-3: Insufficient disk space mid-provisioning → explicit failure with space requirement; retry re-runs cleanly (no corrupted leftover state blocking the next attempt).
- EC-4: The user installs the app over an existing app installation (reinstall/upgrade) → the existing installation is replaced in place; no duplicate app entries or second uninstall record.
- EC-5: Unsupported platform build reaches a user (e.g., wrong architecture) → the app refuses with a clear platform message instead of crashing.

### US-002: Recover from a failed or interrupted first-run provisioning

**As a** desktop-first newcomer, **I want** interrupted provisioning to recover cleanly, **so that** one failure does not leave my machine in a state I cannot fix.

Acceptance criteria:

- AC-1: Given provisioning fails at any stage, when the user retries from the error state, then provisioning resumes or restarts cleanly and can succeed without manual cleanup.
- AC-2: Given the user quits the app mid-provisioning, when the app is reopened, then it detects the incomplete state and offers to continue or start over — never assumes success.
- AC-3: Given provisioning failed, when the user inspects the error state, then a diagnostic detail (reason + where to find logs) is available in-app.

Edge cases:

- EC-1: Two app launches race during first run → only one provisioning flow runs; the second launch focuses the first window (see US-009).
- EC-2: Retry after a transient network failure → succeeds without requiring reinstalling the app.
- EC-3: A runtime appears externally (user installs via CLI channel) while the app sits in a provisioning-error state → on retry/relaunch the app detects and attaches to it instead of provisioning a duplicate.

### US-003: App attaches to my already-running runtime and touches nothing

**As a** CLI power user, **I want** the app to connect to the runtime I already run, **so that** my workspaces, sessions, and in-flight agent work continue exactly as they are.

Acceptance criteria:

- AC-1: Given a healthy runtime is running for the active home, when the app launches, then it attaches to that runtime and renders the same product state the browser would show — no second runtime process appears.
- AC-2: Given the app attached to a runtime it did not start, when the app is used or quit, then the runtime is never stopped, restarted, or modified by the app.
- AC-3: Given the user runs an isolated home configuration, when the app launches, then it resolves the same home the user's environment designates (never a hardcoded location).

Edge cases:

- EC-1: Runtime discovery record exists but the process is dead (stale record) → the app treats the runtime as absent (per US-004/US-001 paths) instead of attaching to a ghost.
- EC-2: A different local process squats the expected runtime address → the app refuses to treat it as CompozyOS and reports the conflict (see US-006).
- EC-3: Runtime is running but unhealthy (responding with errors) → the app shows the degraded state honestly rather than a blank or looping UI.

### US-004: App starts my installed-but-stopped runtime

**As a** CLI power user, **I want** the app to start the runtime I have installed when it is not running, **so that** opening the app is enough to start working.

Acceptance criteria:

- AC-1: Given the runtime is installed but not running, when the app launches, then the app starts it, waits for readiness with visible progress, and lands in the UI.
- AC-2: Given the app started the runtime, when the app quits, then the runtime keeps running (quit never stops the runtime — see US-008).

Edge cases:

- EC-1: The runtime fails to start (corrupt install, port conflict) → the app surfaces the startup failure with the runtime's own error evidence and a retry affordance; it does not loop restarting forever.
- EC-2: Startup takes unusually long (cold disk, antivirus scan) → the app keeps an honest progress state within a bounded window before offering diagnostics; it never shows a dead white screen.
- EC-3: Two actors (app and user's terminal) try to start the runtime simultaneously → exactly one runtime instance results, and the app attaches to it.

## Runtime connection states

### US-005: Incompatible app/runtime versions produce a guided state

**As a** CLI power user, **I want** the app to detect version incompatibility with the runtime, **so that** skew results in guidance instead of subtle breakage.

Acceptance criteria:

- AC-1: Given the attached runtime's version is outside the app's supported range, when the app launches, then it presents an explicit incompatibility state naming both versions and the recommended action (update the app, or update the runtime via its channel), and does not render a broken product UI.
- AC-2: Given versions are compatible, when the app launches, then no compatibility friction is visible.

Edge cases:

- EC-1: Runtime is newer than the app supports → same guided state (recommendation: update the app); never assume forward compatibility.
- EC-2: Version information is unavailable (very old or corrupted runtime) → treated as incompatible with the same guided state.

### US-006: Runtime unavailable states are honest and actionable

**As a** user, **I want** every "cannot reach the runtime" situation to be a designed screen, **so that** I always know what is wrong and what to do.

Acceptance criteria:

- AC-1: Given the runtime address is occupied by a foreign process, when the app launches, then the state names the conflict and the practical resolution; the app never renders the foreign process's content.
- AC-2: Given the runtime cannot be provisioned, started, or attached, when the user reaches the error state, then it always offers: retry, diagnostics (log access), and a path to help — never a raw engine error page.

Edge cases:

- EC-1: Repeated failures across retries → the state escalates detail (shows recent runtime log tail) instead of repeating the same message.
- EC-2: The failure resolves externally (user fixes it in terminal) → retry from the same screen succeeds without app restart.

### US-007: Runtime dying while the app is open degrades gracefully

**As a** user, **I want** a runtime crash to produce a reconnect experience, **so that** I do not lose my place or stare at a dead window.

Acceptance criteria:

- AC-1: Given the app is showing the UI, when the runtime stops responding or exits, then within a short, perceivable interval the app presents a disconnected state (not a frozen or blank UI) offering reconnect and (when the runtime is startable) restart-runtime.
- AC-2: Given the runtime comes back, when reconnection succeeds, then the user returns to the product with current state (whatever the runtime preserved) without restarting the app.

Edge cases:

- EC-1: Crash during an in-progress user action → the action's failure is reported by the UI's existing error handling; the shell adds the disconnected state only when the runtime is actually unreachable.
- EC-2: Rapid crash-restart cycles → the app does not flap; after bounded attempts it settles in the error state with diagnostics (no infinite restart loop).
- EC-3: Network-level interruption only (runtime alive, connection dropped) → reconnect resolves without runtime restart.

### US-008: Closing the app never kills my runtime or agent work

**As a** CLI power user, **I want** quitting the app to be a UI-only event, **so that** long-running agent sessions and background work always survive.

Acceptance criteria:

- AC-1: Given agents/sessions are working, when the user closes the window (the app exits), then the runtime and all in-flight work continue, verifiable via the CLI status surface.
- AC-2: Given the app provisioned and started the runtime, when the app quits, then the same guarantee holds — quit is never a runtime stop.
- AC-3: Given the user wants the runtime stopped, when they use the runtime's own stop surface (CLI), then that remains the one explicit way to stop it.

Edge cases:

- EC-1: OS shutdown/logout while the app is open → the app exits like any app; runtime lifecycle follows the OS's process handling for background services, not an app-initiated stop.
- EC-2: Force-kill of the app process → no runtime damage; next launch attaches normally.

## Window & native integration

### US-009: Launching again focuses the existing window

**As a** user, **I want** a second launch to focus the running app, **so that** CompozyOS never multiplies into duplicate windows the way browser tabs did.

Acceptance criteria:

- AC-1: Given the app is running, when the user launches it again (dock, launcher, installer's "open", file manager), then the existing window is focused/unminimized and no second instance appears.
- AC-2: Given the app is running minimized or on another virtual desktop, when a second launch occurs, then the window is brought to the user's attention per platform convention.

Edge cases:

- EC-1: Second launch with a deep link argument → the existing window receives and navigates to the link (see US-010) — the link is never dropped.
- EC-2: Stale single-instance state after a crash → a fresh launch recovers and starts normally (no "already running" dead end with no running app).

### US-010: CompozyOS links open inside the app

**As a** user, **I want** CompozyOS links (from docs, terminal output, notifications) to open the right view in the app, **so that** following a link never spawns another browser tab.

Acceptance criteria:

- AC-1: Given the app is running, when the user activates a CompozyOS link, then the app window focuses and navigates to the linked view.
- AC-2: Given the app is not running, when the user activates a CompozyOS link, then the app launches and, once ready, lands on the linked view (the link survives the cold start).

Edge cases:

- EC-1: Link targets an entity that no longer exists → the app opens and shows the product's standard not-found experience for that entity, not an error dialog.
- EC-2: Malformed/hostile link payload → rejected safely; the app opens at its default view and never executes or navigates outside the product UI.
- EC-3: Two links activated in quick succession → the last one wins visibly; no queue explosion or double windows.
- EC-4: Link arrives while the app is in a runtime-error state → the app preserves the destination and completes navigation once the runtime is available again.

### US-011: External links open in my default browser

**As a** user, **I want** non-CompozyOS links to leave the app, **so that** the app never becomes an accidental general-purpose browser.

Acceptance criteria:

- AC-1: Given any link to an external destination inside the product UI, when the user activates it, then it opens in the OS default browser and the app window does not navigate away.
- AC-2: Given a UI action attempts to open a new window to external content, when it triggers, then the same rule applies (external → OS browser; the app stays on the product).

Edge cases:

- EC-1: Link that looks internal but points to a non-product origin → treated as external (deny-by-default posture).

### US-012: Window geometry persists and recovers from invalid state

**As a** user, **I want** the window to remember where I put it and recover when displays change, **so that** the app always opens somewhere usable.

Acceptance criteria:

- AC-1: Given the user resizes/moves the window and restarts the app, when it reopens, then position and size are restored.
- AC-2: Given the saved geometry is invalid for the current displays (monitor removed, size below minimum), when the app opens, then it recovers to a sane centered window instead of opening off-screen or unusably small.

Edge cases:

- EC-1: Restore while the previous session ended minimized → the app opens visible (never restores into an invisible state).
- EC-2: Display scale/resolution changed between sessions → restored bounds are validated against current work areas before applying.

### US-013: Launch is flash-free and never hangs on a loading screen

**As a** user, **I want** app startup to look intentional, **so that** the first seconds feel like a product, not a webpage loading.

Acceptance criteria:

- AC-1: Given a normal launch, when the window appears, then there is no white flash and no unstyled intermediate frame; a branded loading state covers the gap until the UI is interactive.
- AC-2: Given the UI fails to become ready within a bounded time, when the timeout elapses, then the app transitions to the honest error/diagnostic state (US-006) — a loading state can never be permanent.

Edge cases:

- EC-1: Very slow machine or cold start → loading state remains visibly alive (progress/animation), and the bounded timeout still applies.
- EC-2: Launch into a runtime-error condition → the branded loading state hands off to the error state without flashing the broken UI.

## App auto-update

### US-014: App updates arrive automatically and apply on my restart

**As a** user, **I want** the app to keep itself current in the background, **so that** I get fixes and features without manual downloads.

Acceptance criteria:

- AC-1: Given a newer app version exists on my channel, when the app has been running (checks happen on launch and periodically, off the critical path), then the update downloads in the background and the app tells me it is ready.
- AC-2: Given an update is ready, when I confirm restart (or restart naturally), then the new version is running afterward and the version indicator reflects it; the app never force-restarts me mid-work without consent.
- AC-3: Given an update artifact fails verification against the product's release signature, when the updater evaluates it, then it is never applied and the app reports the update as unavailable with a diagnostic.
- AC-4: Given no newer version exists, when checks run, then nothing interrupts the user (silent no-op).

Edge cases:

- EC-1: Update check with no network → silent skip; retried later; never an error dialog for a background check.
- EC-2: Download interrupted → resumed or restarted on the next cycle; a partial download is never applied.
- EC-3: User quits with an update downloaded but unapplied → next launch applies it (or launches into it) per platform convention; the update is not lost.
- EC-4: Repeated update failure across cycles → escalates to US-015's fallback rather than looping silently.
- EC-5: Machine sleeps/wakes across a check cycle → checks resume on wake without duplicate prompts.

### US-015: A failed update never strands me silently

**As a** user, **I want** update failures to surface a way forward, **so that** I never sit unknowingly on a stale, unpatchable version.

Acceptance criteria:

- AC-1: Given the apply/restart step fails (the app does not come back on the new version), when the user next interacts with the app, then it reports the failed update and offers a manual download path for the current release.
- AC-2: Given the update feed is malformed or unreachable for an extended period, when the user opens the update status surface, then the app shows last-checked time and the error condition (no pretending everything is fine).

Edge cases:

- EC-1: Update applied but the app crashes on the new version at startup → the situation is detectable (crash evidence available in diagnostics) and the manual-download fallback remains reachable from the OS (the app never becomes unopenable-with-no-path-forward on a healthy machine).
- EC-2: Feed advertises a version older than installed → ignored (no downgrade offer).

## Runtime update surfacing

### US-016: App-provisioned runtime updates feel like one product update

**As a** desktop-first newcomer, **I want** the runtime the app installed for me to stay current through the app, **so that** I never have to learn a second update mechanism.

Acceptance criteria:

- AC-1: Given the app provisioned my runtime and a newer runtime exists on my channel, when the app surfaces updates, then the runtime update appears as part of the app's single update experience.
- AC-2: Given agent work is in flight, when a runtime update is ready to apply, then the app asks for timing consent (apply now vs later) and never restarts the runtime under active work without confirmation.
- AC-3: Given the runtime update applies, when it completes, then the app reconnects automatically and shows the new runtime version.

Edge cases:

- EC-1: Runtime update fails mid-apply → the app reports the failure with diagnostics and the previous runtime remains usable (or the error state guides recovery); the user is never left with no runtime and no explanation.
- EC-2: User declines the update repeatedly → the app keeps working and keeps the update offer available without nagging on every interaction.
- EC-3: The provisioned runtime was later replaced by a managed install (user adopted the CLI channel) → the app detects the ownership change and switches to US-017 behavior.

### US-017: Managed runtime installs get a recommendation, never a write

**As a** CLI power user, **I want** the app to respect my package-manager-owned runtime, **so that** nothing ever mutates binaries behind my manager's back.

Acceptance criteria:

- AC-1: Given my runtime came from a managed channel, when a newer runtime exists, then the app shows the availability plus the exact recommended command/channel to update — and performs no modification itself.
- AC-2: Given I update the runtime through my channel, when the app next connects, then it reflects the new version with no residual "update pending" state.

Edge cases:

- EC-1: Install-method detection is inconclusive → the app defaults to the safe behavior (recommendation only, no write) and says why.

## Channels

### US-018: I can see which release channel I am on

**As a** user, **I want** channel visibility, **so that** I understand what stream of builds I receive.

Acceptance criteria:

- AC-1: Given the v1 app, when the user views the app's about/update surface, then it shows the channel (beta) alongside the version.
- AC-2: Given only beta exists at launch, when the user looks for channel options, then no inactive/empty "stable" choice is offered (reserved identity stays invisible until activation).

Edge cases:

- EC-1: A future stable activation → beta users' transition path is governed by a later decision (explicitly out of v1 scope; see PRD Open Questions); v1 makes no promise in the UI.

## Agent manageability

### US-019: Launch and inspect the desktop surface from the command line

**As an** agent or automation operator, **I want** structured command-line access to the desktop app surface, **so that** the desktop capability is operable without a human or the UI.

Acceptance criteria:

- AC-1: Given the app is installed, when `compozy app open` runs, then the app launches (or focuses, if running) and the command exits with a deterministic success/failure result.
- AC-2: Given any state, when `compozy app status` runs with structured output, then it reports: app installed (and version) or not, app running or not, runtime attach state, and current update state — machine-readable and stable.
- AC-3: Given the app is not installed, when `compozy app` verbs run, then the error is deterministic and names the condition (not-installed) rather than failing obscurely.

Edge cases:

- EC-1: `compozy app open` with a target link/view argument → opens the app at that view (parity with US-010) or returns a deterministic unsupported-argument error if not provided in v1 — behavior must be explicit either way.
- EC-2: Status queried while the app is mid-update or mid-provisioning → the transitional state is reported truthfully (not reduced to running/not-running).

## Browser coexistence

### US-020: Browser and app show the same product with the same state

**As a** CLI power user, **I want** the app and the browser to be two doors to one product, **so that** choosing the app costs me nothing I had in the browser.

Acceptance criteria:

- AC-1: Given I used the product in the browser, when I open the desktop app on the same machine/home, then I see the same workspaces, sessions, and local UI state I had (no re-setup, no lost preferences).
- AC-2: Given both the app and a browser tab are open, when I act in one, then the other reflects the change per the product's normal live-update behavior (no desktop-specific divergence).

Edge cases:

- EC-1: Browser and app pointed at different homes/instances (advanced setups) → each shows its own instance truthfully; the app never silently mixes instances.

## Release integrity

### US-021: An unpublishable release is blocked before users can see it

**As a** release operator, **I want** the desktop release process to fail closed, **so that** I cannot ship an update that installed apps would reject or that would strand the fleet.

Acceptance criteria:

- AC-1: Given release publication is attempted without complete signing material (platform signing or update-feed signing), when the pipeline runs, then it hard-fails before anything reaches the update feed.
- AC-2: Given artifacts are published, when the feed updates, then every published platform entry is verified (artifact reachable, signature valid) as part of the release — a malformed feed entry is a release failure, not a client-side surprise.
- AC-3: Given the update-feed signing key custody procedure, when the first public release ships, then the documented backup and rotation procedure exists and has been reviewed (release gate).

Edge cases:

- EC-1: Partial platform build failure (one platform fails) → the release does not publish a feed that silently drops that platform; the operator gets an explicit choice with evidence.
- EC-2: Publishing to the reserved (inactive) stable channel path before activation → blocked by the pipeline.
