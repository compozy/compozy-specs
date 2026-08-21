# Developer Experience: Skill Sources

Public-surface contract for skill sources. Companion to `_spec.md` (Part II serves this surface) and `_tests.md` (E2E journeys use these exact invocations).

## Golden Path

Machine already has cross-tool skills in `~/.agents/skills` (`frontend-qa`, `git-hygiene`). Fresh CompozyOS install:

```bash
$ compozy skill list
NAME          SOURCE   ORIGIN   DESCRIPTION
compozy       bundled  —        Operate CompozyOS sessions, tasks, and memory
frontend-qa   user     agents   Audit web UIs against the team checklist
git-hygiene   user     agents   Keep branches, commits, and PRs clean
```

The universal folders are on by default — nothing to configure. Enable the Claude convention too, live:

```bash
$ compozy config set skills.sources agents,claude
skills.sources = ["agents", "claude"] (global) · applied live

$ compozy skill list --source user
NAME          SOURCE   ORIGIN   DESCRIPTION
frontend-qa   user     agents   Audit web UIs against the team checklist
git-hygiene   user     agents   Keep branches, commits, and PRs clean
pdf-tools     user     claude   Split, merge, and OCR PDF files
```

`pdf-tools` (from `~/.claude/skills`) now shows up in every session's `/` picker with a `claude` origin chip. Expose a Compozy skill back to the ecosystem:

```bash
$ compozy skill expose review-checklist --to agents
exposed review-checklist → /Users/ana/.agents/skills/review-checklist
```

Claude Code, Codex, Cursor, and every other `.agents/skills` reader now lists `review-checklist`.

## CLI

### `compozy skill sources`

Active sources with daemon-measured state. Honors `--workspace` (ID, name, or path) for workspace-effective view.

```bash
$ compozy skill sources
SOURCE   STATE      WORKSPACE PATH          GLOBAL PATH             SKILLS  NOTES
compozy  always on  .compozy/skills         ~/.compozy/skills       12      —
agents   enabled    .agents/skills          ~/.agents/skills        2       —
claude   enabled    .claude/skills          ~/.claude/skills        1       —
team-skills  enabled  —                     ~/team-skills           47      truncated

scope: global · overrides: none
```

```bash
$ compozy skill sources -o json
{
  "scope": "global",
  "sources": [
    {
      "slug": "compozy",
      "label": "Compozy",
      "kind": "builtin",
      "enabled": true,
      "always_on": true,
      "workspace_path": ".compozy/skills",
      "global_path": "~/.compozy/skills",
      "roots": [
        {
          "root_id": "r_g_builtin_04c9",
          "path": "/Users/ana/.compozy/skills",
          "exists": true,
          "readable": true,
          "scanned_count": 12,
          "skill_count": 12,
          "truncated": false,
          "skipped_links": [],
          "collisions": [],
          "verification": {"blocked": 0, "warned": 0},
          "native_readers": []
        }
      ]
    },
    {
      "slug": "agents",
      "label": "Universal (.agents)",
      "kind": "preset",
      "enabled": true,
      "always_on": false,
      "default": true,
      "workspace_path": ".agents/skills",
      "global_path": "~/.agents/skills",
      "roots": [
        {
          "root_id": "r_g_agents_9f31",
          "path": "/Users/ana/.agents/skills",
          "exists": true,
          "readable": true,
          "scanned_count": 2,
          "skill_count": 2,
          "truncated": false,
          "skipped_links": [],
          "collisions": [],
          "verification": {"blocked": 0, "warned": 0},
          "native_readers": ["openclaw"]
        }
      ]
    },
    {
      "slug": "team-skills",
      "label": "team-skills",
      "kind": "custom",
      "enabled": true,
      "always_on": false,
      "path": "~/team-skills",
      "roots": [
        {
          "root_id": "r_g_custom_a41c",
          "path": "/Users/ana/team-skills",
          "exists": true,
          "readable": true,
          "scanned_count": 51,
          "skill_count": 47,
          "truncated": true,
          "skipped_links": [{"path": "/Users/ana/team-skills/old", "reason": "dangling"}],
          "collisions": [{"name": "audit", "winner_root_id": "r_g_agents_9f31", "qualified_form": "team-skills:audit"}],
          "verification": {"blocked": 1, "warned": 2},
          "native_readers": []
        }
      ]
    }
  ]
}
```

`native_readers` is **per root** (the workspace-scope view shows `["openclaw", "hermes"]` on the workspace `.agents/skills` root — Hermes reads the project-level universal folder but its global home is `~/.hermes/skills`, so the global root lists only `openclaw`). Every root object carries the full diagnostic schema regardless of kind. `"readable": false` (permission denied on an existing directory) omits every count for that root — never zeros.

Workspace-effective view:

```bash
$ compozy skill sources --workspace acme-api
SOURCE   STATE      WORKSPACE PATH   GLOBAL PATH        SKILLS  NOTES
compozy  always on  .compozy/skills  ~/.compozy/skills  9       —
agents   enabled    .agents/skills   ~/.agents/skills   3       —
claude   enabled    .claude/skills   ~/.claude/skills   2       —

scope: workspace (acme-api) · overrides: sources · inherits: custom_sources
```

`-o json` at workspace scope carries `"workspace_id": "ws_01H…"` and the same `"inherits": {"sources": false, "custom_sources": true}` object the settings envelope reports; global scope omits both.

`scanned_count` = SKILL.md candidates found in the root; `skill_count` = winning catalog entries the root contributed. Custom slugs come from the directory basename; a basename collision gets a deterministic short-hash suffix (`team-skills-3f2a`), stable across list reordering. When the runtime is unavailable, counts are omitted entirely — never rendered as zeros.

Exit code 0.

### `compozy config set` / `get` / `unset`

Existing verbs, two new keys. List values accept comma-separated or JSON-array forms; scope follows the existing `--scope` / `--workspace` flags.

```bash
$ compozy config get skills.sources
["agents"]

$ compozy config set skills.sources agents,claude
skills.sources = ["agents", "claude"] (global) · applied live

$ compozy config set --scope workspace --workspace acme-api skills.sources '["agents"]'
skills.sources = ["agents"] (workspace acme-api) · applied live

$ compozy config get --scope workspace --workspace acme-api skills.custom_sources
[] (inherited from global)

$ compozy config set skills.custom_sources '~/team-skills'
skills.custom_sources = ["~/team-skills"] (global) · applied live

$ compozy config unset --scope workspace --workspace acme-api skills.sources
skills.sources unset (workspace acme-api) · inheriting global · applied live
```

Failure — unknown slug:

```bash
$ compozy config set skills.sources agnets
Error: unknown skill source preset "agnets" (did you mean "agents"?) · valid: agents, claude
```

Exit code 1; `-o json` carries `{"error": {"code": "unknown_skill_source", "message": "...", "valid": ["agents","claude"], "suggestion": "agents"}}`.

### `compozy skill expose` / `unexpose`

```bash
$ compozy skill expose review-checklist --to agents
exposed review-checklist → /Users/ana/.agents/skills/review-checklist

$ compozy skill expose review-checklist --to agents
already exposed: review-checklist → /Users/ana/.agents/skills/review-checklist (no change)

$ compozy skill unexpose review-checklist --to agents
unexposed review-checklist ← /Users/ana/.agents/skills/review-checklist
```

Failure invocations (every expose/unexpose failure is the one `expose_failed` envelope; exit code 1):

```bash
$ compozy skill expose review-checklist --to claude
Error: expose failed (1 target)
  claude  expose_target_disabled — source not enabled (enabled targets: agents)

$ compozy skill expose review-checklist --to agents
Error: expose failed (1 target)
  agents  expose_name_conflict — occupied by /repo/.agents/skills/review-checklist

$ compozy skill expose review-checklist --to agents,claude
Error: expose failed (1 of 2 targets; completed targets rolled back)
  agents  rolled_back
  claude  expose_name_conflict — occupied by /repo/.claude/skills/review-checklist
```

`-o json` on failure emits the same envelope the API returns (`error.code: expose_failed`, `results[]`, `rolled_back`). Agent-scope sessions may invoke expose/unexpose — it is a skill operation under the same authorization gate as enable/disable; source configuration stays global/workspace policy.

Multiple targets: `--to agents,claude`. Targets are **enabled presets only** — custom sources are not valid expose targets (a link points at a machine-local path; shared team directories would carry broken links). Workspace-scoped skills link into workspace-level folders; global skills into global-level folders. Failures below in **Errors**.

### `compozy skill create --expose`

```bash
$ compozy skill create review-checklist --expose agents
created .compozy/skills/review-checklist/SKILL.md
exposed review-checklist → /repo/.agents/skills/review-checklist
```

Partial success — the skill is created even when the expose leg fails; exit code 1 states both facts:

```bash
$ compozy skill create review-checklist --expose claude
created .compozy/skills/review-checklist/SKILL.md
Error: expose failed (1 target) — the skill was created; fix the cause and run `compozy skill expose`
  claude  expose_target_disabled — source not enabled (enabled targets: agents)
```

Without `--expose`, behavior is unchanged from today.

### `compozy skill info` / `where`

`info` gains an `EXPOSED TO` block; `where` shows every root that participated, including followed links:

```bash
$ compozy skill info review-checklist
NAME         review-checklist
SOURCE       workspace
DIR          /repo/.compozy/skills/review-checklist
EXPOSED TO   agents → /repo/.agents/skills/review-checklist (healthy)
             claude → /repo/.claude/skills/review-checklist (foreign conflict — not our link; no action)

$ compozy skill where frontend-qa
WINNER   /repo/.compozy/skills/frontend-qa (workspace · compozy)
ALSO     /repo/.agents/skills/frontend-qa (workspace · agents · shadowed — invoke as agents:frontend-qa)
         ~/.agents/skills/frontend-qa (user · agents · shadowed)
LINKS    /repo/.claude/skills/frontend-qa → /repo/.compozy/skills/frontend-qa (exposure · healthy)

$ compozy skill where pdf-tools
WINNER   ~/.claude/skills/pdf-tools (user · claude)
ALSO     — none —
```

### `compozy skill list`

Existing verb; the `ORIGIN` column (and `origin` field in `-o json`) identifies the source that contributed the winning definition: `—` for compozy-native, preset slug, or custom source slug.

## HTTP / UDS API

### `GET /api/settings/skills?scope=global`

Envelope `config` gains the two keys; a daemon-computed `sources` array carries the measured state (same shape as `skill sources -o json` above).

```json
{
  "scope": "global",
  "config": {
    "enabled": true,
    "sources": ["agents", "claude"],
    "custom_sources": ["~/team-skills"],
    "disabled_skills": [],
    "poll_interval": "3s",
    "marketplace": {"registry": "https://marketplace.compozy.com"}
  },
  "sources": [ { "slug": "agents", "kind": "preset", "enabled": true, "roots": [ … ] } ],
  "discovered_count": 62,
  "runtime_available": true
}
```

`?scope=workspace&workspace_id=ws_01H…` returns the workspace-effective view plus `"inherits": {"sources": false, "custom_sources": true}` so clients can render inherited-vs-overridden truthfully.

### `PATCH /api/settings/skills`

Two scope-specific request shapes; scope via query. Responses carry the existing apply metadata (`restart_required: false` — live) and the refreshed `sources` array.

**Global scope** — full-config body as today, plus the two new keys:

```json
// request
{"config": {"enabled": true, "sources": ["agents", "claude"], "custom_sources": ["~/team-skills"], "disabled_skills": [], "poll_interval": "3s", "marketplace": {"registry": "https://marketplace.compozy.com"}}}

// 200 response (apply metadata + refreshed read model)
{"scope": "global", "restart_required": false, "config": { …saved config… }, "sources": [ …refreshed sources[] with per-root counts… ]}
```

**Workspace scope** — presence-aware override object (the tri-state on the wire): field absent = untouched, `null` = clear the override (inherit global), array = set the override.

```json
// set presets override, leave custom_sources untouched
{"override": {"sources": ["agents", "claude"]}}

// return presets to inherited, set custom override
{"override": {"sources": null, "custom_sources": ["./tools/skills"]}}

// 200 response — workspace-effective view with inheritance flags
{"scope": "workspace", "workspace_id": "ws_01H…", "restart_required": false,
 "inherits": {"sources": false, "custom_sources": true},
 "config": { …workspace-effective config… }, "sources": [ … ]}
```

Validation failures (400):

```json
{"error": {"code": "unknown_skill_source", "message": "unknown skill source preset \"agnets\"", "valid": ["agents", "claude"], "suggestion": "agents"}}
```

```json
{"error": {"code": "workspace_scope_field_forbidden", "message": "only sources and custom_sources may be written at workspace scope", "field": "poll_interval"}}
```

### `POST /api/skills/{name}/expose` · `POST /api/skills/{name}/unexpose`

```json
// request
{"targets": ["agents", "claude"], "workspace_id": "ws_01H…"}

// 200 — every target succeeded (workspace_id echoed on workspace-scoped ops; omitted for global skills)
{"name": "review-checklist", "workspace_id": "ws_01H…", "results": [
  {"target": "agents", "ok": true, "exposure": {"target": "agents", "path": "/repo/.agents/skills/review-checklist", "status": "healthy"}},
  {"target": "claude", "ok": true, "exposure": {"target": "claude", "path": "/repo/.claude/skills/review-checklist", "status": "healthy"}}
], "rolled_back": false}

// 409 — ANY failure uses this one envelope (single- and multi-target alike):
// top-level code is always expose_failed; per-target codes carry the detail.
{"error": {"code": "expose_failed", "message": "1 of 2 targets failed; completed targets rolled back"},
 "name": "review-checklist",
 "workspace_id": "ws_01H…",
 "results": [
   {"target": "agents", "ok": false, "error": {"code": "rolled_back"}},
   {"target": "claude", "ok": false, "error": {"code": "expose_name_conflict", "occupied_by": "/repo/.claude/skills/review-checklist"}}
 ], "rolled_back": true}
```

There is exactly one failure shape: `expose_failed` + `results[]` — a single-target failure is the same envelope with one result. Per-target error codes (see **Errors**): `skill_not_exposable`, `expose_target_disabled`, `expose_target_invalid`, `expose_name_conflict`, `expose_link_unsupported`, `expose_foreign_link`, `unsafe_skill_name`. `unexpose` returns the same envelope with per-target independent results (removal is idempotent; no rollback concept).

An enabled preset whose directory doesn't exist yet is a valid target: expose creates the exact preset root (containment-proven) and records it in the operation's rollback set.

`workspace_id` on these routes (and skill detail reads) accepts the **canonical workspace id only**; the CLI accepts ID, name, or path via `--workspace` and resolves before calling. Responses echo the resolved `workspace_id`.

### `GET /api/skills/{name}`

Skill payload gains `"origin": "claude"` (source attribution) and `"exposures": [...]` with `"status": "healthy" | "missing" | "broken" | "foreign_conflict"`:

- `missing` — CompozyOS created this link and nothing is at the path now (re-expose repairs).
- `broken` — the link at the path is **ours** (its literal destination matches our record) but no longer resolves into the skill (unexpose or re-expose allowed).
- `foreign_conflict` — something occupies the path that is not our link; CompozyOS reports it and never touches it.

UDS carries the identical routes and shapes.

## config.toml

```toml
[skills]
# Folder conventions to scan besides Compozy's own (always active).
# Valid presets: "agents" (default on), "claude".
sources = ["agents"]

# Extra skills-only directories. Absolute or ~ paths at global scope;
# workspace configs may also use workspace-relative paths.
custom_sources = []
```

- `skills.sources` — which preset conventions are scanned; applied live.
- `skills.custom_sources` — additional directories scanned as skill roots; applied live.
- Workspace override: the same keys in the workspace's `.compozy/config.toml`; a present key replaces the global list, an absent key inherits.

## Native Tools

No new native tools; three existing tools gain additive fields (descriptors and schema digests refresh in the same change):

- `compozy__skill_list` — each item gains `"origin": "agents" | "claude" | "<custom-slug>" | ""`.
- `compozy__skill_search` — each result gains the same `origin` field.
- `compozy__skill_view` — the skill header gains `origin` and `exposures[]{target, path, status}` (same status vocabulary as the API).

```json
// call
{"name": "review-checklist"}

// result (header excerpt)
{"name": "review-checklist", "origin": "", "enabled": true,
 "exposures": [{"target": "agents", "path": "/repo/.agents/skills/review-checklist", "status": "healthy"}],
 "content": "<skill body>"}
```

Agents change source configuration through the daemon-routed `compozy config set` path shown above and inspect it via `compozy skill sources -o json` (or the settings API).

## Errors

| Condition | Surface | Exact outcome |
|---|---|---|
| Unknown preset slug | CLI/API/UI validation | `unknown_skill_source` — message names the bad slug, `valid` lists slugs, `suggestion` carries the closest match; nothing applied |
| Custom path duplicates an active root | CLI/API/UI validation | `duplicate_skill_source` — names the existing source that already owns the resolved path |
| Workspace-relative path at global scope | CLI/API/UI validation | `invalid_source_path` — "workspace-relative paths require workspace scope" |
| Expose a bundled skill | CLI/API/web | `skill_not_exposable` — "bundled skills have no on-disk home; copy it with `compozy skill create` first" |
| Expose target not enabled | CLI/API/web | `expose_target_disabled` — names the disabled source and the currently enabled targets |
| Expose target names a custom source | CLI/API/web | `expose_target_invalid` — "expose targets are presets; custom sources cannot receive links" |
| Skill name unsafe as a path segment | CLI/API/web | `unsafe_skill_name` — names the offending characters; nothing written |
| Any expose/unexpose failure (single- or multi-target) | CLI/API/web | `expose_failed` — one envelope with per-target results; completed expose targets rolled back (`rolled_back: true`) or per-target cleanup errors when rollback could not finish |
| Target compensated during a multi-target rollback | per-target marker | `rolled_back` — appears as a per-target result code inside the `expose_failed` envelope, never top-level |
| Skill removal blocked by uncleanable exposure link | CLI/API/web | `skill_remove_blocked` — names the failing link; canonical skill preserved; retry after fixing the cause |
| Scan root exists but is unreadable (permission denied) | `skill sources`, settings | root reports `readable: false` with counts omitted — never zeros |
| Unexpose of a link with no ownership record | CLI/API/web | `expose_foreign_link` — refused; names the link's actual target |
| Workspace PATCH touches a non-source field | API/CLI | `workspace_scope_field_forbidden` — names the offending field |
| Target folder already has that name (foreign entry) | CLI/API/web | `expose_name_conflict` — names the occupying path; nothing written |
| Filesystem cannot create the link | CLI/API/web | `expose_link_unsupported` — deterministic message; no copy fallback |
| Skill picked in `/` but its source got disabled before submit | session | existing invocation drift rejection (unchanged shape) |
| Scan root over per-root limits | `skill sources`, settings | `truncated: true` on the root; count reflects the scanned subset |
