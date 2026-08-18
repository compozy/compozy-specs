# Developer Experience: Electron Desktop Shell Migration

Public-surface contract for the desktop shell migration. Companion to `_spec.md` (Part II serves this surface) and `_tests.md` (E2E journeys use these exact invocations).

Parity governs this surface with three user-decided exceptions: (1) **`compozy update` is the single update command** — it covers runtime and app, and `compozy app update` is deleted (ADR-005); (2) **update offers live in the web UI** (Settings + menubar indicator, see `_uiux.md`) and the app's boot overlay keeps only non-interactive phases (ADR-006); (3) a **`blocked` status** reports single-flight refusals. Everything else — `compozy app status/open/retry/diagnose`, config keys, deep links, error codes — is byte-compatible with today.

## Golden Path

Daily developer on macOS, app installed, update day:

```console
$ compozy update -o json
{
  "status": "updated",
  "runtime": {
    "status": "updated",
    "install_method": "desktop-app",
    "managed": false,
    "current_version": "0.5.1",
    "latest_version": "0.5.1",
    "release_url": "https://github.com/compozy/compozy/releases/tag/v0.5.1",
    "daemon_restarted": true,
    "message": "Updated CompozyOS runtime to 0.5.1 and restarted the daemon."
  },
  "app": {
    "status": "updated",
    "running": true,
    "current_version": "0.5.1",
    "latest_version": "0.5.1",
    "message": "CompozyOS app is restarting on 0.5.1."
  }
}
$ echo $?
0
```

Headless agent on a server with no desktop app — same command, same mechanism, `app` object simply absent:

```console
$ compozy update -o json
{"status":"updated","runtime":{"status":"updated","install_method":"direct-binary","managed":false,"current_version":"0.5.1","latest_version":"0.5.1","release_url":"https://github.com/compozy/compozy/releases/tag/v0.5.1","daemon_restarted":true,"message":"Updated CompozyOS runtime to 0.5.1 and restarted the daemon."}}
```

## CLI

### `compozy update` — one command, every applicable target

Default applies everything applicable on this host: the runtime always; the desktop app when installed. `--check` is read-only.

```console
$ compozy update --check
Update
  Status           available
  Runtime          0.5.0 → 0.5.1 (desktop-app)
  App              0.5.0 → 0.5.1 (running)
  Release          https://github.com/compozy/compozy/releases/tag/v0.5.1

$ compozy update --check -o json
{
  "status": "available",
  "runtime": {"status":"available","install_method":"desktop-app","managed":false,"current_version":"0.5.0","latest_version":"0.5.1","release_url":"https://github.com/compozy/compozy/releases/tag/v0.5.1","daemon_restarted":false,"message":"CompozyOS runtime 0.5.1 is available."},
  "app": {"status":"available","running":true,"current_version":"0.5.0","latest_version":"0.5.1","message":"CompozyOS app 0.5.1 is available."}
}
```

App installed but **closed** — the app update stages and applies on next launch:

```console
$ compozy update -o json
{
  "status": "staged",
  "runtime": {"status":"updated","install_method":"desktop-app","managed":false,"current_version":"0.5.1","latest_version":"0.5.1","release_url":"https://github.com/compozy/compozy/releases/tag/v0.5.1","daemon_restarted":true,"message":"Updated CompozyOS runtime to 0.5.1 and restarted the daemon."},
  "app": {"status":"staged","running":false,"current_version":"0.5.0","latest_version":"0.5.1","message":"CompozyOS app update staged; applies on next launch."}
}
$ echo $?
0
```

Package-manager install — recommendation, never mutation:

```console
$ compozy update -o json
{"status":"available","runtime":{"status":"available","install_method":"homebrew","managed":true,"current_version":"0.5.0","latest_version":"0.5.1","release_url":"https://github.com/compozy/compozy/releases/tag/v0.5.1","daemon_restarted":false,"recommendation":"brew upgrade compozy","message":"CompozyOS 0.5.1 is available. Upgrade with your package manager."}}
```

Another update already in flight (app or CLI — single-flight per home):

```console
$ compozy update -o json
{"status":"blocked","runtime":{"status":"blocked","install_method":"desktop-app","managed":false,"current_version":"0.5.0","latest_version":"0.5.1","daemon_restarted":false,"message":"A runtime update is already in progress (holder pid 4242). Retry after it completes."}}
$ echo $?
1
```

Canceling a stuck (dormant) operation — staged app never picked up, or a dead holder:

```console
$ compozy update --cancel -o json
{"status":"canceled","operation_id":"op-7f3a2c","message":"Canceled dormant update operation; the update channel is free."}
$ echo $?
0

$ compozy update --cancel -o json     # a live executor is running — declined
{"status":"blocked","operation_id":"op-7f3a2c","message":"Operation op-7f3a2c has a live executor (pid 4242 via cli). Not canceled."}
$ echo $?
1
```

Rules the examples encode:

- Aggregate `status` precedence: `failed` > `blocked` > `staged` > `updated` > `available` > `up-to-date`. Exit 1 only for `failed` and `blocked`.
- The `app` object exists exactly when a desktop app is installed on this host.
- The old flat record's fields live unchanged inside `runtime` (agents migrate by prefixing).
- Failure always restores the previous binary and says so in `message`; the structured record is emitted before the non-zero exit.
- **One truth:** `compozy update --check`, the daemon's background check, and the web UI Settings section always report the same versions for the same home and channel.

### Removed: `compozy app update` (ADR-005)

```console
$ compozy app update --check
Error: unknown command "update" for "compozy app"
$ echo $?
1
```

Delete targets: the verb, its tests, `docs/cli/app/update` reference page. Scripts migrate to `compozy update`.

### `compozy app status` — verbs and errors unchanged; schema takes its one designed step (v2)

Pure local read; never launches anything; validates against the published app-state schema — now `schema_version: 2` (adds the live-operation fields and new update states; shell and CLI ship the step lockstep, and version mismatches fail with the existing `app_state_schema_unknown` error).

```console
$ compozy app status -o json
{
  "schema_version": 2,
  "installed": true,
  "app_version": "0.5.0",
  "channel": "beta",
  "running": true,
  "pid": 4242,
  "state": "product",
  "runtime": { "attached": true, "owned": true },
  "update": {
    "app_state": "idle",
    "runtime_state": "idle",
    "last_checked_at": "2026-08-20T14:02:11Z"
  }
}
```

Not installed (exit 0 — a state, not an error):

```console
$ compozy app status -o json
{"schema_version":2,"installed":false,"running":false,"runtime":{"attached":false,"owned":false},"update":{"app_state":"idle","runtime_state":"idle"}}
```

### `compozy app open [path]` — unchanged

```console
$ compozy app open /tasks/42
# app running → navigates via the local control channel, exits 0

$ compozy app open /tasks/42        # installed, not running
# → cold start via compozyos://open/tasks/42, exits 0 once handed to the OS

$ compozy app open '../etc'
{"error":{"code":"invalid_target_path","message":"app: target path must be absolute"}}
$ echo $?
1
```

### `compozy app retry` — unchanged

```console
$ compozy app retry -o json      # after a failed boot
{"ok":true}

$ compozy app retry -o json      # app healthy — safe no-op, never corrupts state
{"ok":true}

$ compozy app retry -o json      # app not running
{"error":{"code":"app_not_running","message":"app: control channel is not accepting connections"}}
$ echo $?
1
```

### `compozy app diagnose` — unchanged

```console
$ compozy app diagnose -o json
{
  "schema_version": 1,
  "boot_id": "boot-7f3a",
  "boot_phase": "error",
  "app_version": "0.5.0",
  "runtime_version": null,
  "runtime_owned": null,
  "current_error": { "code": "runtime_unhealthy", "safe_message": "The runtime is not responding." },
  "previous_crash": null
}

$ compozy app diagnose --bundle
{"error":{"code":"diagnostic_consent_required","message":"app: pass --yes to consent to bundling logs and state"}}
$ echo $?
1

$ compozy app diagnose --bundle --yes --bundle-output /tmp/compozy-evidence.tar.gz
Bundle
  Path  /tmp/compozy-evidence.tar.gz
$ echo $?
0
```

## Web UI update surface

Inventory and states in [`_uiux.md`](_uiux.md). Contract-level promises:

- Settings → General → **Updates** shows both tracks (runtime + app) from daemon truth, with an apply action wherever self-apply is possible — identical in the app and in a plain browser.
- A discreet menubar indicator appears only while an update is available and navigates to that section.
- The product UI never invents update state: unknown renders as unknown (SD-007).

## Desktop app overlay (parity reference, reduced roles)

The boot overlay keeps only non-interactive phases, strings frozen as today:

- Boot: phase progress until the product loads.
- Applying: **"Updating CompozyOS"** with staged progress `download` (with %), `verify`, `install`, `start`, `ready`.
- Version skew: **"CompozyOS needs an update"**; managed runtimes show the package-manager recommendation verbatim.
- Error: cause + log path.

The interactive offer ("CompozyOS is ready" + Update button) is deleted (ADR-006); offers live in the web UI.

## Deep links — unchanged

```
compozyos://open                     → default view
compozyos://open/sessions/abc123     → that session
compozyos://open/http://evil.com     → rejected, default view
compozyos://open/../../etc           → rejected, default view
```

## config.toml — unchanged

```toml
[app]
update_check = true            # background checks for app + runtime updates (read-only)
update_check_interval = "6h"   # 15m–168h; out of bounds fails config validation
```

- `update_check = false` → no background checks on either track; manual `--check` still works.
- Interval governs background cadence; CLI checks are always on demand.

## Errors

Frozen `compozy app` taxonomy (envelope `{"error":{"code","message"}}`, exit 1):

| Code | Condition | Points to |
| --- | --- | --- |
| `app_not_installed` | No desktop app registered on this machine | Install page |
| `app_not_running` | Control channel absent/dead within timeout | `compozy app open` (launches) or OS launcher |
| `app_launch_failed` | OS refused to start the app | `compozy app diagnose` |
| `invalid_target_path` | `open` target not an absolute product path | Fix the path argument |
| `app_state_schema_unknown` | Recorded state from an unknown schema version | Update app and CLI to matching versions |
| `app_control_unavailable` | Control transport failed mid-call | Retry; `compozy app diagnose` |
| `diagnostic_consent_required` | `--bundle` without `--yes` | Re-run with `--yes` |

`compozy update` per-target statuses: `up-to-date`, `available`, `updated`, `staged` (app only), `failed`, `blocked`; plus `managed` recommendations that never mutate files. The structured record is always emitted, including on failure. Transient states (`accepted`, `applying`, with phase and percent) are visible while an update runs — in `compozy app status`'s update block, the settings API, and the web section — all reading the same operation truth; the CLI record itself reports the terminal outcome of its own run.
