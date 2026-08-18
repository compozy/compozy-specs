# PRD: CompozyOS Desktop App

## Overview

CompozyOS today is used through the browser, and users tell us it is a bad fit: the product lives as "one more tab" among dozens, loses its identity, and has no native presence — no dock/launcher icon of its own, no single-instance behavior, no OS-level launch story, and no automatic updates. For a product positioned as an agent operating system, arriving as a browser tab undersells it and makes daily use worse.

This PRD defines the **CompozyOS Desktop App**: a native desktop application for macOS, Windows, and Linux that presents the existing product UI in its own window, with native behaviors (single instance, deep links, remembered window state, honest startup/error states) and a complete automatic update system. It is deliberately a **thin shell**: the runtime and the product UI remain the single source of product behavior; the app adds native presence and lifecycle, and nothing else. The product interface itself requires zero changes in v1.

It serves two populations. **Desktop-first newcomers** get a real first-run: download, open, guided setup, working product — with one coherent update experience afterward. **CLI power users** get a native window onto the runtime they already operate, with an ironclad guarantee: the app attaches to what exists, never disturbs it, and quitting the app never kills the runtime or in-flight agent work.

## Goals

Observable outcomes that define success:

- A user with no prior CompozyOS installation reaches a working product from a single download, without using a terminal, on macOS, Windows, and Linux.
- A user who already runs the runtime opens the app and sees their exact product state; the app never starts a duplicate runtime, never stops a runtime it did not start, and never modifies a runtime installed through a managed channel.
- Launching the app when it is already running focuses the existing window — the app can never multiply into duplicate instances the way browser tabs did.
- CompozyOS links open inside the app (running or cold-started); external links leave the app for the default browser.
- The app keeps itself current automatically: updates download in the background and apply on an explicit restart; a failed update always surfaces a way forward; an unverifiable update is never applied.
- Users whose runtime was provisioned by the app receive runtime updates through the same single update experience; users on managed installs receive a correct recommendation instead of a write.
- Closing the window ends the app and nothing else: agent work and the runtime continue, verifiable from the CLI.
- Agents and automation can launch and inspect the desktop surface through the CLI with structured output (`compozy app`), with deterministic errors — the desktop capability is not UI-only.
- Installation on macOS and Windows passes the platforms' publisher-trust checks (no unsigned-software walls).
- Browser access continues to work unchanged; browser and app are two doors to the same product and state.

## User Stories

Full catalog: [Full user stories](_user_stories.md). Feature-area index:

- **US-001–US-004 — Install & first run**: newcomer provisioning, recovery, attach to running runtime, start installed runtime.
- **US-005–US-008 — Runtime connection states**: version-skew guidance, honest unavailable states, graceful degradation on crash, quit-never-stops-runtime.
- **US-009–US-013 — Window & native integration**: single instance, deep links, external links, window-state persistence/recovery, flash-free bounded startup.
- **US-014–US-015 — App auto-update**: background check/download, consent-based restart, verification, failure fallback.
- **US-016–US-017 — Runtime update surfacing**: single update experience for app-provisioned runtimes; recommendation-only for managed installs.
- **US-018 — Channels**: channel visibility (beta at launch).
- **US-019 — Agent manageability**: `compozy app open`/`compozy app status` with structured output.
- **US-020 — Browser coexistence**: same product, same state, both surfaces.
- **US-021 — Release integrity**: fail-closed publication; feed verification; signing-key custody gate.

## Core Features

### 1. Native window over the existing product UI

One native window that hosts the product interface exactly as the runtime serves it. The window carries CompozyOS identity (name, icon), a branded flash-free launch state, remembered geometry with invalid-state recovery, and honest designed screens for every "product not reachable" condition (starting, provisioning, incompatible versions, runtime down, address conflict). The product UI inside is byte-for-byte the same product browser users see; no desktop-specific fork of the interface exists.

### 2. Runtime resolution: attach-first, provision-if-absent

On launch the app resolves a runtime in this order: attach to a healthy running runtime for the active home; start an installed-but-stopped runtime; provision one through a guided first-run flow when none exists. Ownership rules are absolute: the app never stops, restarts, replaces, or modifies a runtime it did not start or install; version compatibility is checked before rendering, and incompatibility is a guided state, not a broken screen.

### 3. Complete automatic update system

The app checks for updates on launch and periodically (off the critical path), downloads in the background, verifies authenticity before anything is applied, and applies on user-consented restart. Failure paths are first-class: a failed apply surfaces a manual-download fallback; a broken or unreachable feed is visible in the update status; downgrades are never offered. Runtime updates follow the ownership split: provisioned runtimes update through the app as one experience (with timing consent when agent work is in flight); managed installs get the correct channel recommendation and are never touched.

### 4. Native integration for the "out of tabs" promise

Single-instance focus on relaunch; a CompozyOS link scheme so docs, terminal output, and notifications open views inside the app (surviving cold start); external links always open in the OS default browser; OS-level launch surfaces (dock, launcher, start menu) carry CompozyOS identity.

### 5. Agent-operable desktop surface

The desktop capability is manageable without the UI: `compozy app open` launches or focuses the app; `compozy app status` reports installed/running state, app version, runtime attach state, and update state in structured output with deterministic errors. Configuration that the app introduces lives in the product's standard configuration file with the standard lifecycle (defaults, validation, docs).

### 6. Release channels

The app launches on a single **beta** channel, matching the product's actual release state. The stable channel's identity (naming, feed location, per-channel app identity) is designed and reserved from day one so graduation is an activation, not a migration. Channel and version are visible in the app.

Feature interactions: runtime resolution (2) gates everything the window (1) shows; the update system (3) reads the ownership established by (2); deep links (4) queue behind runtime readiness (2); the CLI surface (5) reports the states produced by (1)–(3).

## Business Rules

Ownership and lifecycle:

- BR-1: The app never stops, restarts, replaces, or modifies a runtime it did not start or provision. No exception, including during updates.
- BR-2: Quitting the app never stops the runtime — including runtimes the app itself started or provisioned. Stopping the runtime is only ever an explicit action on the runtime's own control surface.
- BR-3: At most one app instance runs per user session; a second launch focuses the first and forwards any link arguments.
- BR-4: The app resolves the runtime home from the user's environment configuration; it never assumes a fixed location.
- BR-5: A runtime discovery record whose process is dead is treated as "no runtime", never attached to.

Updates:

- BR-6: The app applies an update only after authenticity verification succeeds; an unverifiable artifact is never applied, and the failure is reported.
- BR-7: Updates apply only on user-consented restart; the app never force-restarts itself during use.
- BR-8: The app modifies runtime binaries only for runtimes it provisioned (and that remain app-owned); for managed installs it surfaces availability plus the channel-correct recommendation. If ownership is inconclusive, the app defaults to recommendation-only.
- BR-9: A runtime update never applies while agent work is in flight without explicit timing consent.
- BR-10: Version downgrades are never offered by the update system.
- BR-11: App and runtime versions may differ; the app declares a supported runtime-version range and renders the guided incompatibility state outside it (both older-than and newer-than are incompatible).

States and honesty:

- BR-12: Every unreachable-product condition maps to a designed state with a named cause, retry, and diagnostics access; the app never renders a raw engine error page or a foreign process's content.
- BR-13: Loading states are bounded: past the bound, the app transitions to the error state. No permanent spinner.
- BR-14: Startup/restart attempts after failure are bounded; the app never loops restarting a failing runtime.

Links and boundaries:

- BR-15: Only CompozyOS-scheme links navigate the app; every external destination opens in the OS default browser (deny-by-default for anything else).
- BR-16: Malformed or hostile link payloads are rejected safely; navigation lands on the default view.

Channels and release:

- BR-17: v1 publishes to the beta channel only; the reserved stable identity is not user-selectable and not publishable until explicitly activated.
- BR-18: Desktop release publication fails closed: incomplete signing material or a malformed/unverifiable feed entry blocks the release before users can observe it.
- BR-19: Durable app identity (name, identifiers, link scheme, install/uninstall identity) is CompozyOS-derived from the first public build and never changes thereafter.

Agent surface:

- BR-20: Every state the app can be in (not installed, installed-not-running, provisioning, attaching, running, updating, error) is reported truthfully by the structured status surface with deterministic vocabulary.

## User Experience

Personas and goals: see `_user_stories.md` (Personas). Primary flows:

**Newcomer first run:** download installer → OS-trusted install → open → branded welcome → guided provisioning with visible stages → working product UI. Later: one update experience covers app and runtime (US-001, US-002, US-016).

**Existing user adoption:** install app → open → app attaches to running runtime (or starts installed one) → identical product state as the browser, including local UI state (same-surface continuity) → browser remains available; user retires their pinned tab at their own pace (US-003, US-004, US-020).

**Daily loop:** click dock/launcher icon → single window focuses → work → close window when done; agents keep running; links from terminal/docs/notifications reopen the exact view in the app (US-008, US-009, US-010).

**Update moment:** unobtrusive "update ready" indication → user restarts when convenient → new version confirmed in the app; runtime updates ride the same moment for app-owned runtimes, with timing consent when agents are busy (US-014, US-016).

UX considerations: every runtime/connection state is a designed screen with a next action; startup is flash-free with a branded bounded loading state; the app follows each platform's conventions for focus, relaunch, and window behavior; accessibility of the product UI is inherited from the web interface (no desktop-specific regression), and the shell's own screens (loading, errors, provisioning, update prompts) meet the product's existing accessibility floor. Discoverability: the docs' getting-started path presents the desktop app as the primary install for interactive use, with browser and CLI paths intact.

## High-Level Technical Constraints

- **Reuse over rebuild**: the app must present the product UI the runtime already serves, unchanged; v1 requires zero changes to the product web interface. The shell contains only window/lifecycle/update/native-affordance behavior — every product capability stays in the runtime.
- **Runtime is a shared resource**: single-runtime-per-home semantics, discovery, and liveness rules of the existing runtime govern the app's behavior; the app must honor isolated home configurations.
- **Trust requirements**: macOS and Windows builds must be signed (and notarized where the platform requires it) so installation passes platform trust checks and updates can be applied; update artifacts must be cryptographically verified before applying. The signing material already provisioned in the organization is reused; the update-feed signing key is new, with documented custody (backup + rotation procedure) as a launch gate (ADR-004).
- **Update distribution**: the hot update path (feed + artifacts) is served from the organization's own distribution domain with stable URLs, decoupled from code-hosting choices; the code-hosting release page remains the canonical archive (ADR-004). Fine-grained placement is a TechSpec decision.
- **Performance from the user's seat**: launch-to-interactive comparable to opening the product in a browser on the same machine; update checks and downloads never block or degrade product use; second-launch focus feels immediate.
- **Streaming-heavy UI**: the product UI depends on many concurrent live streams; the app must be validated on each platform's native web engine under real streaming load before release (a known cross-engine risk from research).
- **Privacy**: the app introduces no new data collection; diagnostics live in local logs the user can access.
- **Platform floor**: macOS (arm64, x64), Windows x64, Linux x64. Windows arm64 is out (the runtime does not ship it).
- **Agent manageability**: every app capability is operable via CLI with structured output and deterministic errors, per the product's agent-first premise.
- **Naming**: durable identifiers derive from CompozyOS (ADR-006); the app never introduces new user-facing vocabulary conflicting with the product glossary.
- **Extension ecosystem**: the app adds no new extension points and changes none in v1 — extensions, hooks, skills, tools/resources, and registries continue to interact with the runtime, which the app merely presents. Checked surfaces: the shell exposes no programmable surface of its own; anything extensions can reach today they reach identically with the app in front. Shell-level extension points, if ever wanted, are a future decision.

## Non-Goals (Out of Scope)

- **Tray icon / menubar mode / background app lifecycle** — closing the window quits the app (runtime lifecycle is independent). A tray presence changes the lifecycle contract and is a separate future decision.
- **Multiple windows** — v1 is a single-window app.
- **Any rebuild of product UI natively** — no native re-implementation of views, no desktop-specific UI fork.
- **In-app windowing changes** — the product's internal window-manager UX (virtual desktops, tiled windows) is untouched; if part of the "many tabs" pain is internal, that is its own initiative.
- **Replacing browser access** — the browser surface remains fully supported; the app is an addition, not a migration.
- **Brand migration beyond the app** — renaming the rest of the product's surfaces (site, docs, CLI binary) to CompozyOS is a separate initiative (ADR-006); this PRD pins the new name only into the app's durable identity.
- **Stable channel activation** — designed and reserved, not shipped (ADR-005).
- **Mobile, web-store, or OS-app-store distribution** — direct distribution only in v1; app-store constraints (sandboxing) conflict with the runtime model and are out of scope.
- **Remote/multi-instance management UI** — the app targets the local runtime; managing remote instances from the desktop shell is out of v1 (the product's existing remote access feature is unaffected).
- **Windows arm64** — excluded until the runtime ships it.

## Architecture Decision Records

- [ADR-001: Ship a native desktop app as a thin shell over the existing runtime and web UI](adrs/adr-001.md) — thin-shell direction; zero web-UI changes in v1.
- [ADR-002: Hybrid attach-first runtime relationship with guided provisioning](adrs/adr-002.md) — attach > start > provision; never disturb what the app does not own.
- [ADR-003: Update ownership split by installer](adrs/adr-003.md) — app always self-updates; runtime updated by the app only when app-provisioned; managed installs get recommendations.
- [ADR-004: Three-platform launch reusing organization signing credentials; own-domain hot update path](adrs/adr-004.md) — simultaneous macOS/Windows/Linux; reuse provisioned signing material; new update-feed key with custody gate; own-domain feed.
- [ADR-005: Single beta channel at launch, stable identity reserved from day one](adrs/adr-005.md) — honest channel model, graduation-ready.
- [ADR-006: CompozyOS durable app identity; `compozy app` CLI verb](adrs/adr-006.md) — new name in the retrofit-hostile surfaces; non-colliding CLI verb.

Research corpus behind these decisions: `analysis/summary.md` plus the five slice analyses under `analysis/`, and the handoff brief `_brief.md` (this directory).

## Open Questions

- **Brand migration timing**: when does the product-wide Compozy → CompozyOS naming migration (site, docs, prose, glossary, possibly the CLI binary name) run, and who owns it? The app shipping as CompozyOS while prose says Compozy is an accepted temporary inconsistency (ADR-006) — the follow-up initiative needs an owner and a slot.
- **Stable graduation path for beta users**: when the stable channel activates, do existing beta installs stay on beta or move to stable by default? Deferred until graduation planning (ADR-005).
- **Deep-link target coverage in v1**: link-opening is in scope (US-010); the exact set of product views addressable by links in v1 (sessions, workspaces, tasks, settings) needs confirmation during TechSpec against what the product UI can route to today.
- **`compozy app open <target>`**: whether the CLI verb accepts a view/link target in v1 or ships open/status only (US-019.EC-1) — either is acceptable, must be decided explicitly in TechSpec.
- **Org-admin prerequisites scheduling**: granting the new repo visibility to the organization's signing secrets and generating the update-feed keypair are administrative gates on the critical path (ADR-004) — need an owner and a date before release work starts.
- **"Many tabs" residual pain**: after the app ships, validate with users whether the remaining pain is internal window sprawl (product UI concern, separate initiative) — research flagged this split as unresolved.
