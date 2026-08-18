---
type: feature
created: 2026-08-09 15:11
source: compozy/compozy · main · research corpus at .compozy/tasks/desktop-app/analysis/ (5 slices + summary)
---

# Package the existing web UI as a native Tauri V2 desktop app, with a complete auto-update system — without rebuilding anything the daemon already does

## Problem

Users report that running Compozy only in the browser is a bad experience: the product lives as "yet more browser tabs", loses its identity among them, and has no native presence (no dock/taskbar icon, no single-instance focus, no OS-level launch, no auto-update). The outcome wanted is a first-class desktop application for macOS, Windows, and Linux that:

1. Wraps the **existing** `web/` interface (the primary product surface) in a native window — with the **minimum possible effort and minimum new surface area**; nothing is rebuilt inside the shell that the daemon or the SPA already provides.
2. Reuses the **existing Go daemon** as the engine — including the possibility of loading the daemon-served web UI directly inside the shell.
3. Ships a **complete, robust auto-update system using Tauri's own mechanisms** (signed artifacts, update feed, background download, controlled restart).
4. Ends the "many tabs" failure mode: launching the app a second time focuses the existing window; links to Compozy open in the app, not in a new browser tab.

User stories:

- As a Compozy user, I install a desktop app (dmg/installer/AppImage) and get the exact same UI I know from the browser, connected to my existing workspaces, sessions, and agents — nothing is lost or reset.
- As a user who already runs `compozy` from the CLI, the desktop app attaches to my running daemon instead of fighting it, and never kills a daemon it did not start.
- As a user, updates arrive automatically: the app tells me an update is ready, downloads it in the background, and applies it on an explicit restart — and it never bricks or silently fails.
- As an operator/agent, I can launch and inspect the desktop surface through the CLI, and its configuration lives in `config.toml` like every other capability.

## Current State

### How the web UI is served today

- The daemon serves the **production SPA itself**. The built bundle is **not** embedded from this repo: the daemon imports the external Go module `github.com/compozy/compozy-web-assets` (pinned `v0.0.68`, `go.mod:12`) and serves it from memory via a Gin `NoRoute` SPA fallback (`internal/api/httpapi/static.go:33`, `internal/api/httpapi/routes.go:31`). Any extensionless path resolves to `index.html`; `/api`, `/ws` are hard-bypassed (`static.go:161-184`).
- `COMPOZY_WEB_DIST_DIR` swaps the embedded bundle for any directory on disk (`static.go:18,25-30`) — the seam `make dev`/`make start` already exploit (`Makefile:184-193`, `scripts/dev.sh:174-209`).
- The SPA resolves its API base as **`window.location.origin`** — no configurable base URL, no injected runtime config (`web/src/lib/api-client.ts:6-7`). SSE goes through a ticketed `EventSource` that latches into `"local"` mode (no ticket) when `/api/gateway/stream-tickets` 404s (`web/src/lib/gateway-stream-auth.ts:48-60`, `web/src/lib/ticketed-event-source.ts:99-132`). The window-manager stream is a WebSocket (`internal/api/httpapi/window_manager_routes.go:20`).
- The daemon's browser-request protection middleware (Go 1.25 `http.NewCrossOriginProtection()` + bound-host check) only engages for browser-shaped requests and only accepts the request's own origin or the bound-host origin; there is no wildcard and no configurable allowlist (`internal/api/httpapi/middleware.go:59-177`).

### Daemon lifecycle, discovery, auth

- Defaults: `http://localhost:2123` (`internal/config/defaults.go:16`, `config.toml:4-6`). Port `0` (ephemeral) is supported end-to-end (`internal/api/httpapi/server_start.go:68-71`), but there is **no port-conflict fallback** — a busy port is a terminal listen error (`server_start.go:45-56`).
- Discovery: `~/.compozy/daemon.json` `{pid, port, started_at}` written atomically (`internal/daemon/info.go`); singleton `flock` + PID file with stale-PID detection (`internal/daemon/lock.go`); liveness = PID **and** process start-time match (`internal/cli/daemon_status.go:76-93`). `COMPOZY_HOME` relocates the whole layout (`internal/config/home.go:145-158`).
- Lifecycle: `compozy daemon start` is detached by default with readiness polling until the reported PID matches the child (`internal/cli/daemon.go:214-258`); `daemon stop` sends SIGTERM and polls (`daemon.go:114-145`); ordered graceful shutdown (`internal/daemon/daemon_lifecycle.go:146-190`). `compozy status -o json` returns `{status, pid, started_at, http_host, http_port, version}` (`internal/api/contract/status.go:19-33`); `compozy open` already computes the exact web URL and hands it to the OS browser (`internal/cli/open.go:54-62`).
- Auth: the local HTTP surface has **no token/cookie auth** — only browser-protection + loopback guards (`internal/api/httpapi/routes.go:10-33`, `middleware.go:366-388`). The remote gateway feature adds *separate* authenticated listeners, off by default (`internal/config/gateway_config.go:82-102`).
- Version/update surfaces: build metadata via ldflags (`internal/version/version.go`), versioned status contract (`StatusSchemaVersion`, `contract/status.go:7`), and `GET /api/settings/update` returning `{available, latest_version, install_method, managed, recommendation, …}` with managed-install detection (homebrew/npm/apt/…) that deliberately refuses self-update for managed installs (`internal/update/types.go:98-111`, `internal/update/detect.go:67-97`).

### Release and distribution today

- goreleaser-pro, `CGO_ENABLED=0` (static cross-compiles for every platform), matrix `linux|darwin|windows × amd64|arm64` **minus windows/arm64** (`.goreleaser.yml:16-45`). Channels stable/beta computed once by a release-plan step and enforced against the published GitHub Release (`.github/workflows/release.yml:603-700`). Artifacts: archives + deb/rpm + npm `@compozy/cli` + Homebrew tap + hosted `install.sh`; `checksums.txt` signed with cosign (keyless, workflow-pinned identity); SBOMs (`.goreleaser.yml:57-201`). There is **no Apple Developer ID / Windows Authenticode configuration anywhere in the repo today**.

### Prior art in our own history

- A full Tauri v2 desktop effort exists in `~/dev/compozy-code` (branch `tauri`): ~58.6K lines of Rust, ~25 custom plugins, 155 IPC channels, Node SEA sidecar, six-platform signed pipeline, Ed25519 updater on R2. It never launched — phases 5–6 unstarted, every benchmark UNCONFIRMED, Electron deletion FAIL (`packages/tauri/README.md:318-329`, `tasks/prd-tauri/benchmark-results-44.md`, `tasks/prd-tauri-final/review_consolidated_summary.md`). Directly reusable pieces: the trait-seamed updater plugin (`src-tauri/src/plugins/updater.rs`), channel-aware release config generated in CI (`.github/workflows/release.yml:236-262`), the unsigned-updater-payload CI gate, the splash/`splash-ready` handshake with double timeouts, window-state recovery with a denylist, exit-cleanup and supervision constants, per-platform config overlays, dev/prod entitlement and deep-link-scheme splits.

### Naming collision

- `compozy desktop` and `compozy window` are **already shipped CLI verbs** of the daemon's in-browser virtual-desktop/window-manager domain (`internal/cli/root.go:177-178`, docs under `packages/site/content/docs/cli/desktop/`). The native app cannot take the `desktop` name without a hard-cut rename of that tree.

## Evidence

The full research corpus lives at `.compozy/tasks/desktop-app/analysis/` (5 seven-section analyses + `summary.md`). Decisive verbatim facts:

- Tauri config reference: *"when a URL is provided, the application won't have bundled assets"* — and a remote/`http://` origin window gets **no IPC** unless a capability grants `remote.urls`, which Tauri's own docs frame as *"similar to driving a very fast race car without any safety features enabled"* (https://v2.tauri.app/reference/config/, https://v2.tauri.app/security/capabilities/).
- Version baseline (2026-08-09): `tauri` 2.11.5 (2026-07-01), `tauri-plugin-updater` 2.10.1, `tauri-apps/tauri-action@v1`, MSRV 1.77.2 (crates.io / docs.rs).
- Updater sharp edges, all upstream-documented: single `pubkey`, **no key rotation** (open: https://github.com/tauri-apps/tauri/issues/7585); a lost private key = no installed client can ever update again; one malformed platform entry in `latest.json` breaks updates for **all** platforms; Windows NSIS can update the main exe while a bundled sidecar silently stays stale (https://github.com/tauri-apps/tauri/issues/15134); macOS notarization breaks when `externalBin` is added without inside-out signing (https://github.com/tauri-apps/tauri/issues/11992).
- CORS silent-failure precedent: daemon allow-lists `tauri://localhost` and forgets `http://tauri.localhost` → Windows fails with an opaque "Failed to fetch" and **zero server log lines** (https://github.com/block/buzz/issues/3490).
- Reference products: **Synara, T3Code, and Open-Design are all Electron — none uses Tauri** (03 §Overview). They diverge exactly on the decisions Compozy faces: bundle-vs-proxy window strategy, 2-file vs 2600-line vs custom updater. Open-Design's proxy failure catalogue (SSE pool starvation, gzip double-decode, retry amplification) and Electron's `ignore-connections-limit` loopback switch (`open-design/apps/desktop/src/main/index.ts:144-163`) have **no known WKWebView/WebView2 equivalent** — an unmeasured risk for Compozy's SSE-heavy UI.
- Hermes ships an Electron app + a 5–10MB **Tauri bootstrap-installer**, and has **no auto-updater at all** — it re-builds the app from a git checkout on the user's machine, with an enormous incident-driven hardening apparatus around it (02). Its architecture is the cautionary tale for "update by rebuilding"; Tauri's signed-artifact updater is precisely what it lacks.
- The daemon cross-compiles statically today (`CGO_ENABLED=0` in `.goreleaser.yml:16-19`) — so producing per-triple sidecar binaries is cheap *if* bundling is chosen.

## Requirements

### R1 — Window strategy (v1): load the daemon-served UI; keep the shell pure-native

- The main window loads the URL the daemon already serves (the same URL `compozy open` computes today), so that: `web/` requires **zero changes**; API/SSE/WS stay same-origin; existing browser users on `http://localhost:2123` keep their `localStorage`/IndexedDB when they move to the desktop app (same origin = same storage — no reference product ever solved browser→desktop state migration; this sidesteps it entirely).
- The page receives **no Tauri IPC**: no `remote.urls` capability grant, ever, without a dedicated security review. Everything native (updater UX, dialogs, single-instance focus, deep-link routing, window-state) is implemented on the Rust side of the shell.
- The window must render **honest native states** when the daemon is not serving: starting (with progress), port-conflict, incompatible-version, failed (with log access) — never a raw WebView error page.
- Escalation trigger (documented, not implemented): if in-page native affordances become product requirements (in-UI update prompts, native file dialogs from the SPA), the fallback is bundling `frontendDist` with daemon CORS extension to `tauri://localhost` + `http://tauri.localhost` and an injected API base — a fork that then requires the state-migration and auth work v1 avoids. A Rust-side stable-origin proxy is the *last* resort (it re-implements the daemon's HTTP surface and conflicts with "daemon truth wins").

### R2 — Daemon relationship: attach-first, provision-if-absent

- On launch the shell resolves a daemon in this order: healthy daemon for the active `COMPOZY_HOME` (via `daemon.json` + PID/start-time liveness, i.e. the CLI's existing algorithm) → start one (the shell owns only daemons it started) → honest failure state. It must respect `COMPOZY_HOME` (never hardcode `~/.compozy`) and never kill or restart a daemon it did not start.
- Readiness is probed (announced port/discovery record + HTTP status poll with deadline), never assumed from spawn success; crash-restarts use bounded backoff ending in a terminal, user-visible failed state.
- Version handshake before rendering: compare shell version against `daemon.version` + `StatusSchemaVersion` from `/api/status`; out-of-range skew is a first-class UI state with a recommended action, not a crash or a silent degrade.
- **Open product decision (must be resolved in PRD/techspec; default recommendation: attach-first with *no bundled daemon* in v1, requiring/offering the existing CLI install channels for provisioning):** whether the daemon binary is additionally bundled as a Tauri sidecar (`externalBin`) so the app is self-contained. If bundling is chosen, the spec must adopt: per-triple binaries from the existing static build, inside-out macOS signing of the nested binary, a Windows version resource + installer hook against NSIS sidecar staleness (tauri#15134), and an explicit single-owner rule versus `internal/update` and managed installs.
- ~~Quit contract: closing the window quits the shell; a daemon the shell started is stopped via SIGTERM-with-grace (the daemon's own contract); an attached daemon is left running.~~ **SUPERSEDED (ADR-002, PRD BR-2, peer review N-006): quit never stops any runtime — owned or attached. Stopping is exclusively the runtime's own control surface; the only sanctioned stop is inside the consented update transaction (ADR-011).** (Tray/background mode is out of v1 — see Non-goals.)

### R3 — Auto-update: complete and robust, on Tauri's own mechanisms

- `tauri-plugin-updater` with `createUpdaterArtifacts`, minisign signatures, and static `latest.json` feeds ~~on GitHub Releases~~ **SUPERSEDED (ADR-004, peer review N-006): the hot update path (feed + payloads) serves from the organization's own distribution domain (`releases.compozy.com/desktop/<channel>/`); GitHub Releases remains the canonical archive only** — generated in CI per channel: `stable/latest.json`, `beta/latest.json`, derived from the **existing** release-plan channel outputs — one source of truth for channel identity.
- Update UX: check on launch off the critical path + periodic re-check; background download with progress; update applies only on an explicit user-confirmed restart (on Windows the prompt is mandatory because the installer force-exits the app; the `installMode` must be an explicit decision).
- Robustness requirements (each traceable to a documented failure): CI hard-fails when any updater artifact lacks a `.sig` or the signing key env is absent; a watchdog around install-and-restart falls back to a manual-download affordance when the app fails to exit; the install action is only exposed after signature/manifest verification succeeds; an update/DB-migration interlock defines the recovery UX for a half-applied update (daemon owns append-only Goose migrations; boot-time repair is forbidden — recovery asks, never auto-repairs); malformed-feed protection (a bad platform entry must surface as a diagnosable error, and feed generation is validated in CI before publish).
- Key custody: the minisign private key is generated once, stored in the org secret store with a documented backup, and a **rotation procedure is written before the first public release** (single-`pubkey` limitation, tauri#7585 — rotation requires a transitional release and must never be improvised).
- Ownership rule: the Tauri updater owns the **app bundle** only. It must honor the same discipline as `internal/update`'s managed-install detection — an install delivered by a package manager (e.g. future Homebrew Cask) is *managed* and gets a recommendation, not a self-update. Shell and daemon versions may skew; the handshake in R2 bounds it and `GET /api/settings/update` remains the daemon's own update surface.

### R4 — Desktop affordances (v1 scope, all shell-side)

- **Single instance**: second launch focuses/unminimizes the existing window (this is the product's core promise against "many tabs").
- **Deep links**: register ~~`compozy://`~~ **`compozyos://` (SUPERSEDED naming — ADR-009/ADR-012, peer review B-010/N-006)** (with a separate dev scheme so dev and installed builds don't fight for registration); links route into the running window; cold-start links are queued, not dropped.
- **Window-state**: persist and restore geometry, with invalid-state recovery (tiny/off-screen/minimized → recover to a sane centered window) and validation against current displays.
- **No white flash**: window hidden until first paint signal (with a hard timeout so the app can never hang invisible), background color matched to the app theme.
- **External links**: any non-Compozy URL opens in the OS default browser; window-open is deny-by-default.
- **Native menus**: OS-mandated minimum only (macOS app menu / quit / about); the SPA remains the product surface.

### R5 — Platforms and packaging

- macOS arm64 + x64: `.dmg` for install, signed (Developer ID) + notarized; updater artifact hashed **after** notarization/stapling.
- Windows x64: NSIS installer, signed (Azure Trusted Signing or EV); WebView2 provisioned via `embedBootstrapper`; installer GUID/product identity pinned from the first release. Windows arm64 is out (daemon matrix excludes it).
- Linux x64: AppImage (the only Tauri-updater-capable format) for auto-update; `.deb`/`.rpm` builds may ship but update through the package manager, not the Tauri updater. Known webkit2gtk/NVIDIA blank-window remediations documented for support.
- Channel identity (product name, bundle id, install path, uninstall key, URL scheme) is distinct per channel and pinned before the first public build.

### R6 — CI/Release integration

- Desktop artifacts join the **existing** release: same tag, same channel plan, same GitHub Release as the CLI artifacts; fail-closed signing gates (publishing without complete signing secrets hard-fails); cosign/SBOM parity with existing artifacts where applicable; the whole pipeline runs from `release.yml` (a 4-row matrix: mac arm64, mac x64, linux x64, windows x64).

### R7 — Agent-manageability, config, naming

- A CLI verb launches/inspects the desktop surface (per the core premise: every capability manageable via CLI/HTTP/UDS). ~~**Open product decision:** its name — `compozy desktop` collides with the shipped window-manager verb tree; options are a hard-cut rename of that tree or a different verb (e.g. `compozy app`).~~ **DECIDED (ADR-006): the verb is `compozy app`; the window-manager verbs stay unchanged.** "No CLI surface" is not an option.
- Any new configuration (e.g. desktop-shell keys, update channel selection) lives in `config.toml` with defaults, validation, and docs — full config lifecycle.
- The official CompozyOS skill (`skills/compozy/`) and docs gain the desktop surface when it ships.

### R8 — Open product decisions the spec phase must close (with recommended defaults)

1. Daemon bundling: attach-only v1 (recommended) vs sidecar-bundled. Drives R2/R3 details.
2. ~~CLI/app naming vs the `compozy desktop` collision.~~ **DECIDED (ADR-006): `compozy app`.**
3. Channels at launch: stable-only vs stable+beta (recommended: both, mirroring the existing release tracks).
4. Port policy for desktop: keep fixed `2123` (recommended for v1 — it is what preserves browser-user state) vs ephemeral; ephemeral is a config-lifecycle change touching docs and `config.toml`.
5. Whether the "many tabs" pain includes in-app window sprawl (the runtime already models desktops/windows/tabs server-side) — validate with users; a shell alone may not fully resolve the complaint.

## Constraints & Non-goals

- **Zero shell-driven behavior changes to `web/` in v1.** The minimal-effort mandate is a hard constraint; any option requiring SPA behavior changes (API-base injection, auth tickets for a new origin, state migration) is out of v1 scope. **CLARIFIED (ADR-012): the absorbed product-language hard cut updates display strings only.**
- **Thin shell, by rule**: nothing lands in the shell that is not window/lifecycle/update/native-affordance glue; every product capability stays daemon-side (Open-Design's ~18-handler discipline; the legacy repo's 58.6K-line shell is the canonical counter-example). Project file-size caps and skill dispatch apply to all new code.
- **No `remote.urls` capability grant** for the daemon origin without a dedicated security review; no Rust HTTP proxy layer in v1; no CSP ownership split between shell and daemon.
- **Do not reuse legacy update infrastructure**: `releases.compozy.com`/R2 endpoints and the legacy Ed25519 keys from `compozy-code` are unknown-status; the new pipeline uses GitHub Releases and freshly generated keys.
- **Do not adopt Hermes' update model** (git-based detection, on-device rebuild, local re-signing) in any form.
- **The unauthenticated localhost HTTP surface is inherited, not worsened** — but the desktop app makes an always-running daemon more likely; the spec must state this explicitly as an accepted (or separately addressed) posture, not inherit it silently.
- Non-goals v1: tray/menubar background mode; multiple windows; native re-implementation of any UI; Mac App Store distribution (App Sandbox breaks the sidecar/daemon model); Windows arm64; mobile; remote-gateway packaging changes; browser-extension work.
- Greenfield rules apply: no compat shims; if naming renames land (R7), they are hard cuts across code/APIs/docs in one change.

## Verification

- **Update pipeline**: a staging test proves the full loop — install build N, publish N+1 to a fixture feed (local updater fixture server), observe check → download → verify → restart → running N+1; a real-feed acceptance run against a draft GitHub Release before first public ship. CI asserts signature presence/validity of every updater artifact and validates generated `latest.json` (all platforms present, well-formed) before publish.
- **Attach/spawn matrix**: E2E covers attach-to-running-daemon, spawn-when-absent, port-conflict, stale `daemon.json`, version-skew, daemon-crash-while-open, and quit contracts (owned vs attached daemon) — under isolated `COMPOZY_HOME`s per the project's worktree/QA isolation rules.
- **Cross-platform smoke** on real OSes (not only macOS): window lifecycle, single-instance focus, deep-link cold-start and while-running, SSE-heavy screens under WKWebView/WebView2 (measure concurrent stream count; validate no connection-pool starvation), storage persistence across app restarts.
- **QA tracker**: new `untested` content-addressed scenario files under `docs/qa/scenarios/` for install, first-run attach, update, deep-link, and second-launch focus; walked per `qa-execution` before completion.
- Standard gates: `make gate` per task, `make gate-full` at workstream close.

Compozy Impact Audit (initial, to be finalized per task):

- Native tools: no impact expected on `compozy__*` tool IDs/schemas — checked surfaces: toolsets/descriptors are daemon-side and the shell adds no tool; revisit only if a desktop-launch native tool is added.
- Extensibility and hooks: new distribution surface (desktop bundle) + possible new CLI verb + `config.toml` keys; no extension/hook/registry schema changes expected — checked: extensions, hooks, skills/capabilities, MCP sidecars are daemon-side and unchanged; official skill `skills/compozy/` needs an update when the surface ships (R7).
- Workspace data isolation: no new data written by the shell beyond OS-standard app/window state; all workspace-scoped data continues to flow through the daemon's existing HTTP/SSE surfaces; `COMPOZY_HOME` selection must propagate to discovery (R2) — checked: no new store/cache/SSE path is introduced.
- Official Compozy skill: update required at ship time (new surface, new CLI verb, update semantics) — R7.

Web/Docs impact: `packages/site` — installation page (new desktop channel), a new desktop-app getting-started page, `cli/` tree if naming changes (R7), `web-ui.mdx` entry-point wording; `web/` — none in v1 (hard constraint).

## References

Research corpus (read these first — they carry file:line/URL evidence for every claim above):

- `.compozy/tasks/desktop-app/analysis/summary.md` — cross-slice synthesis, convergences/divergences, consolidated risks
- `.compozy/tasks/desktop-app/analysis/01_analysis_tauri-legacy.md` — legacy Tauri effort: reusable updater/CI/window patterns + failure record
- `.compozy/tasks/desktop-app/analysis/02_analysis_hermes-desktop.md` — Hermes shell↔backend contract, readiness ladder, update anti-pattern
- `.compozy/tasks/desktop-app/analysis/03_analysis_ref-desktops.md` — Synara/T3Code/Open-Design comparison, proxy failure catalogue, updater cost curve, cheapest-pattern judgment
- `.compozy/tasks/desktop-app/analysis/04_analysis_compozy-runtime.md` — every integration point in this repo, with file:line anchors
- `.compozy/tasks/desktop-app/analysis/05_analysis_tauri-v2-web.md` — Tauri v2 mechanics: loading options, sidecar, updater end-to-end, CI/signing, pitfalls (all URLs cited)

Key internal anchors: `internal/api/httpapi/static.go` · `internal/api/httpapi/middleware.go` · `internal/daemon/info.go` · `internal/daemon/lock.go` · `internal/cli/daemon.go` · `internal/cli/open.go` · `internal/api/contract/status.go` · `internal/update/` · `web/src/lib/api-client.ts` · `web/src/lib/ticketed-event-source.ts` · `.goreleaser.yml` · `.github/workflows/release.yml` · `config.toml`

Key external: https://v2.tauri.app/plugin/updater/ · https://v2.tauri.app/develop/sidecar/ · https://v2.tauri.app/security/capabilities/ · https://github.com/tauri-apps/tauri-action · https://v2.tauri.app/distribute/sign/macos/ · https://v2.tauri.app/distribute/sign/windows/ · tauri issues #15134, #7585, #11992 · https://github.com/block/buzz/issues/3490

Legacy reference (patterns, not code): `~/dev/compozy-code` (branch `tauri`) — `packages/tauri/src-tauri/src/plugins/updater.rs`, `.github/workflows/release.yml:236-262,526-599`, `packages/tauri/loading.html`, `tasks/prd-tauri/architecture-decisions.md`
