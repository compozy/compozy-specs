---
status: completed
title: "Update system: provisioning, runtime apply, app auto-update, control socket"
type: desktop
complexity: critical
---

# Task 3: Update system: provisioning, runtime apply, app auto-update, control socket

## Overview

Delivers the complete update capability — the safety core of the product: first-run provisioning from the signed runtime manifest, the home-scoped runtime-mutation transaction (lock + journal + durable ownership), quiesce-gated runtime updates with migration-aware recovery, the hardened app auto-updater, and the `app.sock` control socket that makes every update primitive agent-operable end-to-end with Task 2's CLI clients. This is the task where the fail-closed invariants (6–9, 17–20) become code.

<critical>
- ALWAYS READ the PRD, the TechSpec, and their catalogs (`_user_stories.md`, `_tests.md`) before starting
- REFERENCE TECHSPEC for implementation details — do not duplicate here
- FOCUS ON "WHAT" — describe what needs to be accomplished, not how
- MINIMIZE CODE — show code only to illustrate current structure or problem areas
- TESTS REQUIRED — implement every test case assigned in ## Tests
</critical>

<requirements>
1. MUST verify the minisign signature of `runtime.json` over canonical bytes BEFORE trusting any field; archive `sha256` is bound inside the signed manifest; missing `schema_heads` ⇒ refusal (ADR-011.5, B-001/B-005).
2. MUST run every stop→swap→start inside the `$COMPOZY_HOME/update.lock` transaction (flock + PID+start-time, stale detection, post-acquire re-probe) with the fsync'd phase journal; crash recovery classifies pre-swap rollback vs post-migration `recovery_required` — the old binary is never silently restarted after migrations may have advanced (ADR-011.6/8).
3. MUST reacquire app-provisioned ownership from provenance marker + executable path/hash — never from memory or `app.json`; disagreement ⇒ recommendation-only (BR-8).
4. MUST stop only through the daemon quiesce contract (drain → settle → safe-to-stop → revalidate immediately before SIGTERM → undrain on abort); the shell never reads `task_runs` (ADR-011.7).
5. Staging order is fixed: download + verify complete BEFORE any quiesce/stop (single guarantee; N-003).
6. App updater MUST: run its check BEFORE daemon resolution on every launch (crash-on-launch mitigation); take its own `.app` backup around `install()` on macOS and restore on failure (#3505) ; collect Windows consent before install and persist intent via `on_before_exit` (#3514 — "install started" is the last signal in released plugin versions); surface feed 404/`ReleaseNotFound` as a feed-error state, never "up to date" (US-015.AC-2); wire the two-pubkey rotation fallback; honor `update_check=false`; apply only on consented restart with the watchdog → manual-download fallback.
7. MUST implement `$COMPOZY_HOME/app.sock` (0600, versioned envelope): `navigate`, `update.check`, `update.apply(app|runtime)`, `retry`, `diagnose` — the exact primitives the boot-window UI calls; Task 2's CLI clients complete end-to-end here (IT-031).
8. First-run guided provisioning MUST be stage-visible and resumable (US-001/US-002), abort-to-attach when a runtime appears externally, and honest offline refusal.
9. All consent/update/recovery screens render in the `boot` window with DESIGN token values; every state lands in `app.json` truthfully.
</requirements>

## Subtasks

- [x] 3.1 `runtime/artifacts.rs`: signed manifest client (minisign verify → schema_heads → platform entry → sha256)
- [x] 3.2 `runtime/provision.rs`: download→verify→stage→rename pipeline, provenance marker, resume/abort-to-attach, guided boot-window flow
- [x] 3.3 `runtime/mutation.rs`: `update.lock` transaction, phase journal + crash recovery, durable ownership verification
- [x] 3.4 `runtime/quiesce.rs`: drain-contract client with revalidation + undrain compensation
- [x] 3.5 `update/runtime_update.rs`: consent (BR-9), ownership-split UX, apply sequence, `recovery_required` sticky state
- [x] 3.6 `update/app_update.rs`: check-before-daemon ordering, cadence + off-switch, hardened install flows per platform, rotation fallback, watchdog
- [x] 3.7 `control.rs`: `app.sock` server + protocol; boot-window UI and socket converge on the same primitives
- [x] 3.8 End-to-end wiring with Task 2 CLI verbs (`compozy app update|retry|diagnose`) + fixture updater feed with staging keys

## Implementation Details

Rust signatures and file split per TechSpec §Architectural Boundaries / §Core Interfaces; protocol sequencing per ADR-011 decisions 1–8; updater mechanics + live-defect workarounds per `analysis/06_analysis_tauri-core-verified.md` §4; feed/manifest layout + BR-10 posture per `analysis/07_analysis_tauri-dist-release.md` §§4–6b. The quiesce readout shape comes from Task 2's contract change. Fixtures: local updater feed server + staging minisign keypair (never production), fake daemon honoring drain + SIGTERM.

### Relevant Files

- `desktop/src-tauri/src/runtime/{artifacts,provision,mutation,quiesce}.rs`, `desktop/src-tauri/src/update/{app_update,runtime_update}.rs`, `desktop/src-tauri/src/control.rs` — new
- `desktop/tests/` — stub drain surface, fixture feed, fake daemon extensions
- `internal/daemon/lock.go` — the flock+stale-PID pattern `update.lock` mirrors (read-only)
- `internal/update/github.go`, `internal/update/verify.go` — checksum/naming conventions the manifest generation must match (read-only; feed generation itself is task_05)
- `internal/cli/app_control.go` (task_02) — client side of IT-031

### Dependent Files

- `app.json` schema (task_01) — gains truthful `updating`/`recovery_required` transitions exercised here
- task_05 release pipeline — generates the `runtime.json`+`.minisig` this task consumes (schema fixed HERE, generation there)

### Related ADRs

- [ADR-003](adrs/adr-003.md) — ownership split
- [ADR-008](adrs/adr-008.md) — provisioning + provenance
- [ADR-011](adrs/adr-011.md) — decisions 1–8 (this task's contract)
- [ADR-004](adrs/adr-004.md)/[ADR-005](adrs/adr-005.md) — feed origin + channel layout consumed

## Deliverables

- Newcomer first-run: download→verify→install→start→product, resumable, honest offline state
- App-provisioned runtime updates: consent → quiesce → locked swap → restart → reconnect; managed installs get recommendations; recovery state machine live
- App auto-update loop against the fixture feed incl. all hardening paths
- `app.sock` + CLI verbs operating every primitive headlessly
- Every test case assigned in `## Tests` implemented and passing **(REQUIRED)**

## Tests

- [x] UT-029–UT-038 — signed manifest + provisioning pipeline + atomicity + platform refusal
- [x] UT-056–UT-064 — app updater orchestration, verification failure, cadence, watchdog, off-switch
- [x] UT-065–UT-070 — runtime updater ownership split, consent, apply failure, decline
- [x] UT-092–UT-095 — update.lock contention/stale/re-probe + ownership hash mismatch
- [x] UT-096–UT-098 — manifest signature/key/schema_heads refusals
- [x] UT-100–UT-102 — journal phases, crash classification, sticky recovery
- [x] UT-108 — quiesce revalidation race
- [x] UT-109, UT-110 — control-socket protocol + permissions
- [x] UT-114, UT-115 — macOS backup/restore hardening; 404 ≠ up-to-date
- [x] IT-003, IT-011, IT-012 — full provisioning, interruption resume, transient network retry
- [x] IT-013–IT-015 — updater loop, tampered artifact, malformed feed
- [x] IT-016–IT-018 — owned apply end-to-end, verify-before-stop guarantee, managed recommendation
- [x] IT-025 — off-switch observable (zero feed hits)
- [x] IT-029, IT-030 — migration-interlock recovery journey, quiesce race end-to-end
- [x] IT-031 — CLI ⇄ app.sock round trip

## Web/Docs Impact

None in this task (docs land in task_04; support-runbook content sourced from this task's behaviors). **QA impact:** flag only — update/provisioning/recovery are user-visible; scenarios planned in task_06, walked in task_07.

## Extensibility / Agent Manageability / Config Lifecycle

- Extensibility: none — no extension surface touched. Checked: manifests, hooks, provide surfaces, registries, MCP.
- Agent manageability: completes it — every update primitive reachable via `compozy app` + `app.sock` with deterministic errors (B-006 closed).
- Config lifecycle: consumes `[app]` (no new keys); update off-switch behavior tested here (IT-025).

## References

- `analysis/06_analysis_tauri-core-verified.md` §4 (updater corrections + live defects), §6 (spawn)
- `analysis/07_analysis_tauri-dist-release.md` §§4–6b (feed semantics, 404-as-silence, BR-10/roll-forward, rotation)
- `.resources/foundry/packages/desktop/` (updater wiring reference)

## Success Criteria

- Every assigned test case implemented and passing
- Full fixture-feed loop green on Linux/Windows CI; forced-failure paths leave the previous app/runtime intact and diagnosable
- `compozy app update --check/--apply` drives the real flows headlessly with schema-valid output
