# Developer Experience: Profiles

Public-surface contract for Profiles. Companion to `_spec.md` (Part II serves this surface) and `_tests.md` (E2E journeys use these exact invocations). Everything below is written as-if-shipped; undecided values are surface-grill questions, not placeholders.

## Golden Path

```console
$ compozy profile create marketing --color "#FF7F3A" --icon megaphone
Created profile marketing — now active.

$ compozy exec "Draft the launch tweet thread" --format text
… (runs as usual; the session belongs to marketing)

$ compozy session list
NAME                 STATUS   WORKSPACE   UPDATED
launch-tweet-thread  active   my-saas     just now
# only marketing's sessions — every listing is scoped to the active profile

$ compozy session list --all-profiles
PROFILE     NAME                 STATUS   WORKSPACE   UPDATED
marketing   launch-tweet-thread  active   my-saas     just now
default     fix-login-bug        idle     my-saas     2h ago

$ compozy profile use default
Active profile for workspace my-saas: default.

$ compozy profile current
default (remembered choice of workspace my-saas)
```

Updating an existing install: everything you already had — work, config, credentials, memory — belongs to `default`. Nothing moves, nothing asks.

## Resource Folders

Profiles add two folder layers to the two that exist today. All four sum; the most specific layer wins a name conflict. Drop a folder in and it appears on the next catalog refresh — no registration step, same as today.

```text
~/.compozy/agents/writer/                        # user layer — every profile, every workspace
~/.compozy/profiles/dev/agents/writer/           # profile layer — dev only, every workspace
<my-saas>/.compozy/agents/writer/                # project base — committed; every profile, in my-saas
<my-saas>/.compozy/profiles/dev/agents/writer/   # project per-profile — committed; profiles named "dev", in my-saas
```

The winner and what it shadowed are always inspectable:

```console
$ compozy agent list
NAME     LAYER          SHADOWS
writer   project·dev    profile (dev), user
```

Repo folders naming a profile you don't have stay dormant with a hint (see `compozy workspace inspect` and the workspace surface): `declares content for profile "dev" — create it?`.

## CLI

### Selection

`--profile <name>` is a root flag on every command; `COMPOZY_PROFILE` is its environment twin. Precedence: `--profile` → `COMPOZY_PROFILE` → the resolved workspace's remembered choice → `default`. Machine commands (`daemon`, `doctor`, `update`) ignore all of it.

```console
$ COMPOZY_PROFILE=marketing compozy session list   # this terminal acts as marketing
$ compozy task list --profile dev                  # one command as dev, nothing sticks
```

### `compozy profile list`

```console
$ compozy profile list
  NAME        SYMBOL  STATE     WORK
  default     ●       active    12 items
* marketing   📣      active    3 items
  finance     🏦      archived  41 items

$ compozy profile list -o json
[
  {"name":"default","color":"#8E8EB5","icon":"circle","emoji":null,"state":"active","current":false,"work_items":12},
  {"name":"marketing","color":"#FF7F3A","icon":"megaphone","emoji":null,"state":"active","current":true,"work_items":3},
  {"name":"finance","color":"#D6A647","icon":null,"emoji":"🏦","state":"archived","current":false,"work_items":41}
]
```

Exit 0. `*` marks the profile resolved for this invocation (`current` in JSON is that same decoration, not stored state). `work_items` counts what the profile owns at the work roots (sessions, tasks, loop runs, automations, bridges, worktrees, …) — children are never double-counted.

### `compozy profile current`

```console
$ compozy profile current
marketing (remembered choice of workspace my-saas)

$ compozy profile current -o json
{"profile":"marketing","source":"remembered","workspace":"my-saas"}
```

`source` is one of `flag`, `env`, `remembered`, `session`, `default`. An optional `note` explains fallbacks — e.g. `{"profile":"default","source":"default","note":"archived_remembered_fallback","workspace":"my-saas"}`. Exit 0.

### `compozy profile use <name>`

Sets the remembered choice for the resolved workspace (or the Global lens when no workspace resolves).

```console
$ compozy profile use marketing
Active profile for workspace my-saas: marketing.

$ compozy profile use finance
Error: profile "finance" is archived — unarchive it first: compozy profile unarchive finance
```

Exit 0 / exit 1 with `profile_archived`.

### `compozy profile create <name> [--color <hex>] [--icon <name> | --emoji <char>]`

```console
$ compozy profile create marketing --color "#FF7F3A" --icon megaphone
Created profile marketing — now active.

$ compozy profile create Marketing
Error: invalid profile name "Marketing" — use lowercase letters, digits, and hyphens, starting with a letter (e.g. "marketing").

$ compozy profile create all
Error: "all" is reserved (aggregate view). Reserved names: default, all, global.
```

Skipped identity fields are auto-assigned. Creation activates the new profile for the current lens (the API's optional `activate` field). Exit 0 / exit 1 with `profile_name_invalid`, `profile_name_taken`, `profile_name_reserved`.

### `compozy profile update <name> [--color] [--icon | --emoji]`

```console
$ compozy profile update marketing --emoji 🚀
Updated marketing.
```

Works on archived profiles too. Exit 0 / exit 1 with `profile_not_found`.

### `compozy profile rename <old> <new> [--repos all|none]`

```console
$ compozy profile rename dev eng
Renamed dev → eng. Work, config, and credentials follow the profile.
Machine folders renamed: ~/.compozy/profiles/eng/
Repo folders matching "dev" found in 2 workspaces:
  my-saas    .compozy/profiles/dev/   → rename with: compozy profile rename dev eng --repos all
  side-app   .compozy/profiles/dev/
Left as-is, that content is dormant until a profile named "dev" exists again.
Extension placements now dormant: none.

$ compozy profile rename dev eng --repos all
Renamed dev → eng. Repo folders renamed in my-saas, side-app — commit those changes when ready.
```

`--repos` defaults to interactive listing in a TTY and `none` otherwise. Exit 0 / exit 1 with `profile_name_taken`, `profile_permanent` (renaming `default`).

### `compozy profile archive <name>` / `unarchive <name>`

```console
$ compozy profile archive finance
Archived finance. Its work stays visible under "All profiles".
Paused automations (2): invoice-cron, statement-watch.
Queued work frozen with the profile: 3 runs (claimable again after unarchive).

$ compozy profile archive finance
finance is already archived.

$ compozy profile archive marketing
Error: marketing has 1 running session (launch-tweet-thread) — stop it first.

$ compozy profile unarchive finance
Unarchived finance. Paused automations (2) stay paused — reactivate them explicitly:
  compozy automation enable invoice-cron
```

Exit 0 (idempotent notice on repeat) / exit 1 with `profile_sessions_running`, `profile_permanent`.

### `compozy profile delete <name> [--yes]`

```console
$ compozy profile delete scratch
scratch owns no work. This removes permanently:
  agents: 1 (test-writer)   skills: 2   loops: 0   mcp servers: 1
  config overrides: 1 key   credential overrides: none   memory: 12 entries   desktops: 1 saved arrangement
Delete? [y/N] y
Deleted scratch. The name is free.

$ compozy profile delete finance
Error: finance owns 41 work items — deleting work is not supported. Archive instead:
  compozy profile archive finance
```

Exit 0 / exit 1 with `profile_owns_work`, `profile_permanent`.

### Aggregate listings

Every listing verb accepts `--all-profiles`; rows gain the owner column (structured output gains `profile_name`).

```console
$ compozy session list --all-profiles -o jsonl
{"kind":"workspace_resolution","workspace":"my-saas","source":"cwd"}
{"profile_name":"marketing","name":"launch-tweet-thread","status":"active",…}
{"profile_name":"default","name":"fix-login-bug","status":"idle",…}
```

`--profile` and `--all-profiles` together is an error (`profile_selection_conflict`).

## Per-Profile Credentials

The user credentials (your existing keys) serve every profile by default. A profile overrides per provider, in the vault only:

```console
$ compozy profile use marketing
$ compozy secret set providers/openai/api_key
Enter value: ••••••••
Saved for profile marketing (vault:profiles/marketing/providers/openai/api_key).

$ compozy provider inspect openai
openai
  key source : profile marketing (override)     # "user (default)" when no override exists
  auth mode  : bound_secret

$ compozy secret rm providers/openai/api_key
marketing has 3 work items using openai — future runs fall back to the user key. Remove? [y/N] y
Removed. Next runs resolve the user key.
```

The process environment is refused for profile secrets (it is shared by the whole machine):

```console
$ compozy secret set providers/openai/api_key --from-env OPENAI_KEY
Error: profile secrets must live in the vault — the process environment is shared by every profile.
```

Native provider logins (Claude, Codex) stay shared across profiles in v1; `compozy provider inspect` says so plainly.

## Extension Enablement Per Profile

Install is one per machine; each profile chooses what is on (new profiles: everything on).

```console
$ compozy profile use finance
$ compozy extension disable acme/growth-kit
Disabled acme/growth-kit in profile finance. Other profiles keep it on.

$ compozy extension enable acme/growth-kit --profile marketing
Enabled acme/growth-kit in profile marketing.

$ compozy extension list
NAME              INSTALLED   ENABLED (finance)
acme/growth-kit   yes         no
```

## Notification Presets Per Profile

The preset library is shared; each profile chooses what is on.

```console
$ compozy notification-preset list
NAME            ENABLED (marketing)
daily-digest    yes
loud-failures   no

$ compozy notification-preset disable daily-digest          # acts on the active profile
Disabled daily-digest in profile marketing.

$ compozy notification-preset enable loud-failures --profile finance
Enabled loud-failures in profile finance.
```

## HTTP / UDS API

Every route is served identically over HTTP and the local socket; examples show the HTTP form.

Profile identity and lifecycle:

```
GET    /api/profiles                     → 200 [{"name":"marketing","color":"#FF7F3A","icon":"megaphone","emoji":null,"state":"active","work_items":3}, …]
POST   /api/profiles                     {"name":"marketing","color":"#FF7F3A","icon":"megaphone","activate":{"scope":"workspace","workspace_id":"01J9…"}} → 201 {profile}
GET    /api/profiles/ops                 → 200 [{"id":"op_01J…","kind":"rename","profile":"eng","status":"finalizing","step":"2/3"}]
POST   /api/profiles/ops/op_01J…/retry   → 200 {"status":"done"}
PATCH  /api/profiles/marketing           {"emoji":"🚀"}                                                    → 200 {profile}
POST   /api/profiles/dev/rename          {"new_name":"eng","repos":["my-saas"],"plan_revision":"pl_8f2…"}  → 200 {"renamed":true,"repo_results":[{"workspace":"my-saas","renamed":true}],"dormant_placements":[]}
POST   /api/profiles/finance/archive     {"plan_revision":"pl_a11…"}                                       → 200 {"state":"archived","paused_automations":["invoice-cron","statement-watch"],"frozen_queued_runs":3}
POST   /api/profiles/finance/unarchive   {}                                                                → 200 {"state":"active","paused_automations":["invoice-cron","statement-watch"]}
DELETE /api/profiles/scratch?plan_revision=pl_c07…   → 200 {"deleted":true,"removed":{"agents":1,"skills":2,"loops":0,"mcp_servers":1,"config_keys":1,"credential_overrides":0,"memory_entries":12,"desktop_partitions":1}}
```

Rename, archive, and delete **require** `plan_revision` (rename/archive in the JSON body; DELETE as a query parameter). Fetch the matching plan first; a stale revision fails `409 profile_plan_stale`. The CLI does this for you — each verb fetches its plan, shows it, and submits the revision in one flow (also under `--yes` and non-TTY).

`activate` on create is optional: operator surfaces send it (creation activates the lens it was made in — that's why the CLI says "now active"); extension declared-creation never sends it. A profile whose lifecycle operation hasn't finished is **unavailable** everywhere (`profile_unavailable`) until `GET /api/profiles/ops` shows it `done`; `compozy profile ops` lists pending/failed operations and `compozy profile ops retry <op-id>` re-runs a failed one.

Failures use the standard error payload with the codes from the Errors table, e.g. `DELETE /api/profiles/finance` → `409 {"error":{"code":"profile_owns_work","message":"finance owns 41 work items…","action":"POST /api/profiles/finance/archive"}}`. Remote/paired surfaces receive `403 profile_remote_management_forbidden` on every profile-state write — the mutations above, `PUT /api/profiles/selection`, and enablement writes; reads (scoped or labeled aggregate) are all a remote tier gets.

Selection (the remembered choice):

```
GET /api/profiles/selection?workspace_id=01J9…   → 200 {"scope":"workspace","workspace_id":"01J9…","profile":"marketing"}
GET /api/profiles/selection?scope=global         → 200 {"scope":"global","profile":"default"}
PUT /api/profiles/selection                      {"scope":"workspace","workspace_id":"01J9…","profile":"marketing"} → 200 (same shape)
```

Plans — what a rename/archive/delete will do, before it does it (the dialogs and TTY confirmations read these; mutations then carry `plan_revision`):

```
GET /api/profiles/dev/rename-plan?new_name=eng → 200 {"revision":"pl_8f2…","machine_folders":["~/.compozy/profiles/dev/"],"repo_candidates":[{"workspace":"my-saas","path":".compozy/profiles/dev/"}],"dormant_placements":[],"vault_ref_rewrites":2}
GET /api/profiles/finance/archive-plan         → 200 {"revision":"pl_a11…","running_sessions":[],"leased_runs":0,"queued_runs_to_freeze":3,"automations_to_pause":["invoice-cron","statement-watch"]}
GET /api/profiles/scratch/delete-plan          → 200 {"revision":"pl_c07…","removed":{"agents":1,"skills":2,"loops":0,"mcp_servers":1,"config_keys":1,"credential_overrides":0,"memory_entries":12,"desktop_partitions":1},"selections_to_sweep":1}
GET /api/profiles/selection                    → 200 [{"scope":"workspace","workspace_id":"01J9…","profile":"marketing"},{"scope":"global","profile":"default"}]
```

A stale `plan_revision` fails with `409 profile_plan_stale` — re-fetch the plan and confirm again.

Extension enablement:

```
PUT /api/extensions/acme%2Fgrowth-kit/enablement   {"profile":"finance","enabled":false} → 200 {"profile":"finance","enabled":false}
GET /api/extensions/acme%2Fgrowth-kit/enablement   → 200 [{"profile":"default","enabled":true},{"profile":"finance","enabled":false},…]
```

Notification-preset enablement (same shape):

```
GET /api/notifications/presets?profile=marketing                → 200 [{"name":"daily-digest","enabled":true},{"name":"loud-failures","enabled":false}]
PUT /api/notifications/presets/daily-digest/enablement          {"profile":"marketing","enabled":false} → 200
```

Scoped reads (uniform across work listings — sessions, tasks, loops, automations, worktrees, usage):

```
GET /api/sessions?workspace_id=…                  → default profile's rows (no profile param = default)
GET /api/sessions?profile=marketing               → marketing's rows
GET /api/sessions?all_profiles=true               → every profile's rows; each row carries "profile_name"
GET /api/sessions/catalog-stream?workspace_id=…&profile=marketing   → live updates filtered by the daemon
GET /api/sessions/catalog-stream?all_profiles=true                  → labeled aggregate stream
```

`profile` + `all_profiles` together → `400 profile_selection_conflict`. An unknown/archived `profile` value → `404 profile_not_found` / `409 profile_archived` (fail-closed: never a silent fallback). Single-item reads follow the same two modes: a scoped `GET /api/sessions/{id}` returns 404 for another profile's item; `GET /api/sessions/{id}?all_profiles=true` returns it owner-labeled — that labeled read is what deep-link banners use.

## Extension Manifest

An extension places resources per profile and declares profiles that are created at install:

```toml
# compozy-extension.toml (excerpt)
[[profiles]]
name  = "growth"
color = "#5FBF85"
icon  = "chart-line"

[profiles.defaults]          # persona defaults seeded at creation only
agent = "growth-analyst"

[[profiles.credentials]]     # asks surfaced as needs-setup until filled
provider = "openai"
slot     = "api_key"

[[resources.skills]]
path = "skills/tweet-writer"
profile = "growth"           # name-bound placement; omit for every profile

[[resources.skills]]
path = "skills/changelog-writer"   # unplaced: visible in every profile
```

Install experience:

```console
$ compozy extension install acme/growth-kit
acme/growth-kit will:
  • create profile growth (needs 1 credential: openai api_key)
  • add 1 skill to growth, 1 skill to every profile
Proceed? [y/N] y
Installed. Profile growth created — needs setup: openai api_key
  compozy profile use growth && compozy secret set providers/openai/api_key
```

The needs-setup state is durable, profile-owned runtime state (not a manifest read): `compozy profile list` shows a `needs_setup` flag and `GET /api/profiles/{name}` lists each requirement with its missing/filled status; setting the vault secret is what clears it — surviving restarts, extension updates, and even uninstall (the requirement describes the profile).

```console
```

Creation happens once per installed extension; archiving or deleting `growth` is final for this install (boot, updates, and enable/disable never re-create it). A profile named `growth` that already existed is bound, never modified. A full uninstall + fresh install is a new installation and may create it again.

## config.toml

No new keys. Two new **layers** join the existing precedence (most specific wins):

```
defaults → ~/.compozy/config.toml (user) → ~/.compozy/profiles/<p>/config.toml
        → <ws>/.compozy/config.toml → <ws>/.compozy/profiles/<p>/config.toml
```

A profile layer file carries context keys only:

```toml
# ~/.compozy/profiles/marketing/config.toml
[defaults]
agent    = "copywriter"
provider = "openai"
sandbox  = "local"

[hooks]
# profile-layered hooks run only while marketing is active
```

Writes target the layer that owns the current context — the user file under `default`, the profile file under any other profile; `--scope user|profile|workspace` overrides:

```console
$ compozy profile use marketing
$ compozy config set defaults.agent copywriter
Saved to profile marketing (~/.compozy/profiles/marketing/config.toml).

$ compozy config set defaults.agent planner --scope user
Saved to user config — but not applied here: profile marketing sets defaults.agent.

$ compozy config set http.port 3000
Error: http.port is daemon identity — profile layers cannot change it.
Allowed here: defaults.*, hooks.*, permissions.*, … Run with --scope user to change the daemon.
```

## Native Tools

```
compozy__profile_list    {}                     → {"profiles":[{"name":"marketing","state":"active","current":true,…},…]}
compozy__profile_current {}                     → {"profile":"marketing","source":"session","workspace":"my-saas"}
```

Read-only in v1; lifecycle mutations go through the CLI/API surfaces above. Inside a session, `source` is always `"session"` — the binding is immutable (US-028).

## Errors

| Code | Condition | Message points to |
| --- | --- | --- |
| `profile_not_found` | name doesn't exist | `compozy profile list` |
| `profile_archived` | acting as / selecting an archived profile | `compozy profile unarchive <name>` |
| `profile_name_invalid` | grammar violation | the name rule + example |
| `profile_name_taken` | an active/archived profile — or a pending lifecycle operation — holds the name | the holder (profile + state, or the reserving op id) |
| `profile_name_reserved` | `default`, `all`, `global` | the reserved list |
| `profile_permanent` | archive/delete/rename on `default` | nothing — states permanence |
| `profile_owns_work` | delete with owned work | `compozy profile archive <name>` |
| `profile_sessions_running` | archive with running sessions | the running session names |
| `profile_config_key_denied` | daemon-identity key on a profile layer | allowed prefixes + `--scope user` |
| `profile_secret_env_forbidden` | env ref for a profile secret | the vault path to use |
| `profile_selection_conflict` | `--profile` with `--all-profiles` | pick one |
| `profile_plan_stale` | mutation carries an outdated plan revision | re-fetch the plan and confirm again |
| `profile_unavailable` | profile has a pending/failed lifecycle operation | `compozy profile ops` (+ retry) |
| `profile_session_conflict` | acting profile differs from the session's binding | drop the flag/env — the session decides |
| `profile_remote_management_forbidden` | any profile-state write from a remote surface (lifecycle, selection, enablement) | manage locally |
| `profile_deliveries_in_flight` | archive attempted while a notification delivery holds its permit | retry after the delivery settles |

Every CLI failure exits 1 and, under `-o json`, emits the same `{"error":{"code","message","action"}}` payload the API returns.
