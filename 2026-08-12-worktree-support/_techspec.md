# TechSpec: Native Worktree Support

Companion artifacts: `_prd.md` (behavior authority) · `_user_stories.md` (US-001..US-033) · `_uiux.md` + `docs/design/opendesign/worktree/` (web visual contract, S1–S16) · `_tests.md` (test contract) · `adrs/adr-001..011.md`.

**MVP boundary:** this spec designs the complete PRD v1 scope as the MVP — creation, discovery/adoption, nested navigation and selection, parent-brain inheritance, task worktree policy including `per_run` and fan-out isolation, loop environment control, assisted exit with the `forge.provider` surface and the bundled GitHub extension, two-step removal, reconcile-never-cascade, bootstrap contract, configuration, agent surface, and lifecycle events. `cy-create-tasks` numbers the tasks; the trailing `qa-report` + `qa-execution` pair closes the program. Post-MVP (explicitly out of this spec): automation jobs/triggers worktree mode, archive/restore, bulk cleanup, GitHub device-flow OAuth, disk-size measurement, a worktree directive on the loop `fan-out` node itself, non-GitHub forge extensions, live transcript-carrying session fork.

## Executive Summary

A worktree becomes a first-class nested sub-context of its parent workspace: a new `worktrees` table in the global store plus nullable `worktree_id` binding columns on `sessions` and `task_runs` (ADR-007), operated by a new `internal/worktree` package that owns the domain and a hardened, capturing git runner serialized per repository (ADR-008). Sessions extend their containment boundary from "workspace root" to "workspace root ∪ the bound attached worktree's root", evaluated against the registry at all four existing gates (ADR-005). The task execution profile gains a `WorktreePolicy` mirroring `SandboxPolicy` across all 11 established layers, with `per_run` materializing a fresh worktree synchronously inside worker-session preparation (ADR-009). Loops replace the raw `cwd` field with a single `Environment` control on agent-executing nodes. The assisted exit is core git-local logic plus a new `forge.provider` extension surface — GitHub ships bundled, authenticating via extension secret binding with opportunistic `gh` CLI fallback (ADR-010). Branch namespace `run/` (configurable), readable generated names, and compare-and-delete reclamation close the naming contract (ADR-011).

The primary trade-offs: shelling out to system git (truth parity with the user's git, at the cost of a capability-gated binary dependency); tombstone rows over cascade deletion (history preservation at the cost of state filtering everywhere); and synchronous per-run materialization (bootstrap latency inside run start, in exchange for a failure domain equal to the run's own lifecycle and zero orphans).

## System Architecture

### Component Overview

| # | Component | Kind | Responsibility |
|---|---|---|---|
| C1 | `internal/worktree` | new package | Worktree entity, store access, creation/adoption/discovery/status/removal/reconcile, bootstrap, naming, exit engine, git runner + porcelain parsers, per-repo lock, forge consumer interface |
| C2 | `internal/store/globaldb` | modified | `worktrees`, `worktree_status`, `worktree_forge_status` tables; `sessions.worktree_id`, `task_runs.worktree_id`; profile worktree columns; sqlc queries; Goose migration |
| C3 | `internal/session` | modified | `CreateOpts.Worktree`, worktree-aware cwd containment (create + resume), sandbox root selection, worktree binding persisted in session metadata |
| C4 | `internal/task` | modified | `WorktreePolicy` block in `ExecutionProfile`: type, normalize, validate, defaults |
| C5 | `internal/daemon` | modified | Composition: worktree service wiring, `applySessionWorktreePolicy`, per-run materialization call, reuse fingerprint, boot recovery, native worktree tools, forge provider adapter |
| C6 | `internal/loop` + `internal/loop/dsl` | modified | `EnvironmentSpec` on `run-agent`/`goal`, `cwd` input alias normalization, bind-time resolution, linter reason codes |
| C7 | `internal/hooks` + `internal/events` | modified | `worktree` hook family — `pre_create`/`pre_remove` sync-eligible (deny with reason) + `created`/`adopted`/`removed` observe-only — plus canonical event registry entries |
| C8 | `internal/api/{contract,core,httpapi,udsapi}` | modified | Worktree payloads, `BaseHandlers` worktree methods, HTTP+UDS route parity, per-worktree SSE stream, catalog stream, OpenAPI |
| C9 | `internal/cli` | modified | `compozy worktree` verb group; `compozy session new --worktree/--new-worktree` |
| C10 | `internal/tools/builtin` | modified | `compozy__worktree_{list,create,inspect,remove}` + toolset; `worktree` inputs on `session_create`, `task_execution_profile_set`, `task_fanout_runs`, loop tools |
| C11 | `internal/config` | modified | `[worktrees]` section (full lifecycle) + `task.orchestration.profile.default_worktree_mode` |
| C12 | `internal/extensionprotocol` + `internal/extension` | modified | `forge.provider` provide surface, service methods, consumer registry, conformance |
| C13 | `extensions/forge/github` | new | Bundled GitHub forge extension (secret binding + `gh` fallback) |
| C14 | `web/src/systems/{workspace,session,tasks,loops,os}` | modified | Per `_uiux.md` component plan (S1–S16): worktree data layer, nested nav, create/adopt/remove dialogs, environment selectors, policy fieldsets, detail/exit surface |
| C15 | `packages/site` | modified | Worktrees docs section, config/CLI/API references, extension forge docs |

Data flow: web/CLI/tools → HTTP/UDS → `api/core` handlers → `worktree.Service` → git runner + store → hooks/events → SSE streams → web caches. Task runs: claim → `task_session_bridge` → `applySessionWorktreePolicy` → `worktree.Service` (resolve/materialize) → `session.CreateOpts` → ACP subprocess in the worktree directory.

### PRD goal / story → component map

| PRD area (stories) | Owning components |
|---|---|
| Creation US-001..002 | C1 (create, naming, bootstrap), C2, C8, C9, C10, C14-S4 |
| Session env US-003..005 | C3, C8 (session create contract), C14-S7/S8/S9, C9 (`session new` flags), C10 (`session_create`) |
| CLI parity US-006 | C9 |
| Agent tools US-007, US-031 | C10, C5 |
| Discovery/adoption US-008..009 | C1 (discovery, adopt), C8, C14-S1/S2/S3 |
| Navigation US-010..012 | C8 (list payload + catalog stream), C14-S1/S2/S3/S5 |
| Parent brain US-013 | C3 (parent workspace resolution preserved), C5, memory unchanged (directory-scoped) |
| Task policy US-014..016 | C4, C5, C2, C10, C14-S10/S11 |
| Loop env US-017..018 | C6, C14-S12/S13 |
| Assisted exit US-019..025 | C1 (status/exit), C12, C13, C8, C14-S6/S14/S16 |
| Removal/reconcile US-026..028 | C1 (remove/reconcile/dismiss), C7, C14-S15 |
| Bootstrap US-029 | C1 (bootstrap), C11, C7 |
| Configuration US-030 | C11 |
| Extensibility US-032..033 | C12, C13, C7 |

## Architectural Boundaries

- **`internal/worktree` imports**: `internal/store/globaldb` (repository), `internal/fileutil`, `internal/config` (typed config structs only), `internal/hooks` + `internal/events` (dispatch at call sites), `internal/procutil`-style helpers as needed. It defines the `ForgeProvider` interface it consumes; the daemon injects the extension-backed implementation.
- **`internal/worktree` must NOT import** `internal/session`, `internal/task`, `internal/loop`, `internal/workspace`, `internal/daemon`, `internal/api/*`, `internal/cli`. Workspace facts it needs (registry id, root dir) arrive as plain values or via a narrow resolver interface defined in `worktree` and implemented in `daemon`.
- **`internal/session`, `internal/task`, `internal/loop` must NOT import `internal/worktree`.** Each defines the narrow interface it consumes (session: worktree-root resolution for containment; task: ref validation for the policy; loop: environment resolution at bind) and `internal/daemon` injects `worktree.Service`-backed implementations. This keeps the import graph flowing downward with `daemon/` as the sole multi-importer (composition-root discipline, SD-008).
- **`internal/api/core`** imports `internal/worktree` directly (established pattern: handlers import domain packages); HTTP and UDS only register the shared `BaseHandlers` methods.
- **`internal/registry/gitsrc` is untouched** — two git runners with opposite trust contracts stay separate (ADR-008).
- **`extensions/forge/github`** imports only `internal/extensionprotocol` contract types (same rule as `extensions/dev-cycle`).
- `magefiles/boundaries.go` gains the `internal/worktree` rules in the same commit that creates the package.
- **File split (500-line cap, decided up front)** for `internal/worktree`: `worktree.go` (entity/states/errors/Store), `service.go` (struct + options), `naming.go`, `git_runner.go`, `git_porcelain.go`, `lock.go`, `capability.go` (git presence/version gate), `create.go`, `bootstrap.go`, `adopt.go`, `discovery.go`, `status.go`, `exit_plan.go`, `exit_actions.go`, `remove.go`, `reclaim.go`, `reconcile.go`, `dismiss.go`, `events.go`, `forge.go` (consumer interface + types).

## Implementation Design

### Core Interfaces

```go
// internal/worktree/worktree.go
type State string  // pending | ready | failed | missing | removed | dismissed
type Origin string // manual | per_run | adopted

type Worktree struct {
	ID, WorkspaceID, Name, Branch, Path string
	State State
	Origin Origin
	SetupState string // none | ok | failed
	SetupError string
	BaseRef, CreatedHead, RunID string
	CreatedBranch bool
	CreatedAt, UpdatedAt time.Time
}
```

```go
// internal/worktree/service.go — the single mutation/read authority for worktrees
func NewService(store Store, runner GitRunner, opts ...Option) *Service

// Every read/mutation is workspace-predicated: a format-valid wt_ id is never
// authorization proof — the store filters every operation by (workspaceID, id),
// and a mismatch returns worktree_not_found with no existence leak (B-001).
func (s *Service) Create(ctx context.Context, workspaceID string, o CreateOptions) (*Worktree, error) // durable phased creation; returns pending row
func (s *Service) CancelCreate(ctx context.Context, workspaceID, id string) error       // pending only; rolls back artifacts, deletes the row
func (s *Service) Adopt(ctx context.Context, workspaceID, path string) (*Worktree, error)
func (s *Service) List(ctx context.Context, workspaceID string, refresh bool) (*Listing, error)
func (s *Service) Get(ctx context.Context, workspaceID, ref string) (*Worktree, error)  // ref = wt_ id | name
func (s *Service) Status(ctx context.Context, workspaceID, id string, refresh bool) (*Status, error)
func (s *Service) ExitPlan(ctx context.Context, workspaceID, id string) (*ExitPlan, error)
func (s *Service) RunExitAction(ctx context.Context, workspaceID, id string, req ExitActionRequest) (opID string, err error)
func (s *Service) CancelExitAction(ctx context.Context, workspaceID, id, opID string) error // compare-and-cancel on the exact op (B-005)
func (s *Service) Remove(ctx context.Context, workspaceID, id string, force bool) (*RemovalRefusal, error) // refusal non-nil ⇒ blocked with reasons
func (s *Service) Dismiss(ctx context.Context, workspaceID, id string) error
func (s *Service) MaterializeForRun(ctx context.Context, workspaceID string, req RunWorktreeRequest) (*Worktree, error) // thin wrapper over the SAME phased creation primitive as Create (L-004); req carries task/run identifiers ONLY — never a claim token (B-007)
func (s *Service) RecoverCreations(ctx context.Context) error                           // boot recovery from recorded phases; never infers readiness from directory existence
```

```go
// internal/worktree/git_runner.go — injectable; fake in unit tests, real git in integration
type GitRunner interface {
	Run(ctx context.Context, dir string, args ...string) (stdout, stderr []byte, err error)
}
// Real runner: execabs.LookPath("git"); floor git >= 2.37; per-call timeout;
// Stdin=nil; env passthrough MINUS GIT_DIR/GIT_WORK_TREE/GIT_INDEX_FILE,
// PLUS GIT_TERMINAL_PROMPT=0, GCM_INTERACTIVE=never. User config preserved (ADR-008).
```

```go
// internal/worktree/forge.go — consumed here, implemented by the daemon's extension adapter
type ForgeProvider interface {
	Capabilities(ctx context.Context, remoteURLs []string) (*ForgeCapabilities, error)
	Status(ctx context.Context, req ForgeStatusRequest) (*ForgeStatus, error)
	CreatePR(ctx context.Context, req ForgePRRequest) (*ForgePRResult, error)
}
```

```go
// internal/session — defined where consumed; daemon injects the worktree-backed impl
type SessionWorktreeResolver interface {
	// Resolves an attached, ready worktree of the workspace; deterministic errors:
	// worktree_not_found | worktree_not_ready | worktree_missing.
	ResolveSessionWorktree(ctx context.Context, workspaceRegistryID, ref string) (id string, root string, err error)
}
// CreateOpts gains: Worktree string (ref). Bound sessions persist worktree_id in
// session metadata; resume re-resolves and re-validates the SAME binding.
```

```go
// internal/task/profile.go
type WorktreeMode string // inherit | none | ref | per_run
type WorktreePolicy struct {
	Mode        WorktreeMode `json:"mode"`
	WorktreeRef string       `json:"worktree_ref,omitempty"`
}
// ExecutionProfile gains: Worktree WorktreePolicy `json:"worktree"`
// Normalize: empty ⇒ inherit. Validate: ref ⇔ non-empty WorktreeRef; ref must
// resolve (via injected validator) to an attached worktree of the task's workspace.
```

```go
// internal/loop/dsl/node_params.go — replaces RunAgentParams.CWD; added to GoalParams
type EnvironmentSpec struct {
	Mode        string `json:"mode"`                   // root | worktree | per_run | directory
	WorktreeRef string `json:"worktree_ref,omitempty"` // mode=worktree
	Directory   string `json:"directory,omitempty"`    // mode=directory; template-interpolable
}
// HARD CUT (B-009, grill 2026-08-12): params.cwd no longer exists in the DSL.
// A definition carrying it fails validation with a lint reason naming the one-line
// migration (params.cwd → environment.directory); nothing executes with the legacy
// field. US-018 EC-1 amended accordingly. The source and resolved forms carry ONLY
// Environment. Node setting wins over the loop-level default; loop default over root.
```

Error handling convention: every refusal is a sentinel error in `internal/worktree` mapped to one deterministic wire code (table below); wrapping uses `%w`; callers match with `errors.Is/As`.

### Data Models

Declarative fragment `internal/store/globaldb/schema/definitions/xx_worktrees.sql` (next free ordinal) + Goose migration `00060_worktrees.sql` (or next free number at implementation time) + `atlas.sum` + sqlc regen via `make codegen`.

```sql
CREATE TABLE worktrees (
  id            TEXT PRIMARY KEY,             -- wt_<16-hex>, minted at create/adopt (ADR-007)
  workspace_id  TEXT NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE, -- parent registry id
  name          TEXT NOT NULL,                -- record label; UI never shows basenames
  branch        TEXT NOT NULL DEFAULT '',     -- last-known checkout branch; '' = detached
  path          TEXT NOT NULL,                -- canonical absolute path; mutable metadata, never identity
  git_dir       TEXT NOT NULL DEFAULT '',     -- canonical admin gitdir (<common-dir>/worktrees/<name>), recorded at create/adopt; immutable identity fingerprint for restore + leftover validation (B-006)
  state         TEXT NOT NULL CHECK (state IN ('pending','ready','failed','removing','missing','removed','dismissed')),
  pending_phase TEXT NOT NULL DEFAULT '' CHECK (pending_phase IN ('','branch','checkout','copy','setup')), -- durable creation phase driving idempotent boot recovery (B-003)
  origin        TEXT NOT NULL CHECK (origin IN ('manual','per_run','adopted')),
  setup_state   TEXT NOT NULL DEFAULT 'none' CHECK (setup_state IN ('none','ok','failed')),
  setup_error   TEXT NOT NULL DEFAULT '',     -- readable bootstrap failure (US-029 AC-3); redaction-scrubbed before persist (B-007)
  base_ref      TEXT NOT NULL DEFAULT '',     -- recorded base intent; first link of the PR base chain
  created_branch INTEGER NOT NULL DEFAULT 0,  -- 1 = branch minted by Compozy (reclamation gate 1)
  run_namespace TEXT NOT NULL DEFAULT '',     -- branch namespace recorded at creation; immutable reclamation-eligibility fact — config changes never re-classify (ADR-011, B-008)
  created_head  TEXT NOT NULL DEFAULT '',     -- commit the branch was cut at (compare-and-delete value)
  run_id        TEXT NOT NULL DEFAULT '',     -- owning run when origin='per_run'
  created_at    TEXT NOT NULL,
  updated_at    TEXT NOT NULL,
  UNIQUE (workspace_id, name),                -- collision refusal US-001 EC-1
  UNIQUE (workspace_id, id)                   -- composite-FK target: bindings prove same-workspace structurally (B-001)
);
CREATE INDEX idx_worktrees_workspace_state ON worktrees(workspace_id, state);
CREATE UNIQUE INDEX idx_worktrees_live_path ON worktrees(path) WHERE state IN ('pending','ready','removing');

CREATE TABLE worktree_status (               -- last-known git snapshot; NULL = unknown, never zero (BR-13)
  worktree_id  TEXT PRIMARY KEY REFERENCES worktrees(id) ON DELETE CASCADE,
  branch       TEXT,                          -- from `status --porcelain=v2 --branch`
  detached     INTEGER,                       -- 1 = pinned at head_sha (US-012 EC-1)
  head_sha     TEXT,
  dirty_files  INTEGER, insertions INTEGER, deletions INTEGER,
  has_upstream INTEGER, ahead INTEGER, behind INTEGER,
  read_error   TEXT NOT NULL DEFAULT '',      -- non-empty = explicit error state blocking exit actions
  refreshed_at TEXT                           -- staleness marker for every consumer
);

CREATE TABLE worktree_forge_status (         -- network-derived cache; workspace-scoped via worktree FK
  worktree_id TEXT PRIMARY KEY REFERENCES worktrees(id) ON DELETE CASCADE,
  provider    TEXT NOT NULL DEFAULT '',       -- extension name that answered
  pr_number   INTEGER, pr_state TEXT,         -- open | closed | merged
  pr_url      TEXT NOT NULL DEFAULT '',
  merged      INTEGER,
  fetched_at  TEXT                            -- stale marking per BR-14
);

CREATE TABLE worktree_exit_ops (             -- durable exit-operation fence + restart recovery (B-005)
  op_id        TEXT PRIMARY KEY,              -- op_<16-hex>
  workspace_id TEXT NOT NULL,
  worktree_id  TEXT NOT NULL,
  action       TEXT NOT NULL CHECK (action IN ('commit','commit_push','push','open_pr')),
  state        TEXT NOT NULL CHECK (state IN ('running','completed','failed','canceled')),
  started_at   TEXT NOT NULL,
  finished_at  TEXT,
  FOREIGN KEY (workspace_id, worktree_id) REFERENCES worktrees(workspace_id, id) ON DELETE CASCADE
);
CREATE UNIQUE INDEX idx_worktree_exit_ops_active ON worktree_exit_ops(worktree_id) WHERE state = 'running'; -- at most one active op per worktree, enforced in SQL

-- Bindings are composite FKs: a session/run can only reference a worktree of its OWN workspace (B-001).
-- sessions and task_runs already carry workspace_id; the Goose migration performs the SQLite
-- table rebuild required to add the composite constraint to existing tables.
ALTER TABLE sessions  ADD COLUMN worktree_id TEXT;   -- + FOREIGN KEY (workspace_id, worktree_id) REFERENCES worktrees(workspace_id, id)
ALTER TABLE task_runs ADD COLUMN worktree_id TEXT;   -- + FOREIGN KEY (workspace_id, worktree_id) REFERENCES worktrees(workspace_id, id)
CREATE INDEX idx_sessions_worktree  ON sessions(worktree_id)  WHERE worktree_id IS NOT NULL;
CREATE INDEX idx_task_runs_worktree ON task_runs(worktree_id) WHERE worktree_id IS NOT NULL;

-- Enqueue-time policy snapshot: written in the enqueue transaction, consumed by the session
-- bridge ONLY after ClaimNextRun — the live profile is never re-read post-claim (B-002).
ALTER TABLE task_runs ADD COLUMN resolved_worktree_mode TEXT NOT NULL DEFAULT ''
  CHECK (resolved_worktree_mode IN ('','none','ref','per_run'));
ALTER TABLE task_runs ADD COLUMN resolved_worktree_ref TEXT NOT NULL DEFAULT '';

ALTER TABLE task_execution_profiles ADD COLUMN worktree_mode TEXT NOT NULL DEFAULT 'inherit'
  CHECK (worktree_mode IN ('inherit','none','ref','per_run'));
ALTER TABLE task_execution_profiles ADD COLUMN worktree_ref TEXT NOT NULL DEFAULT '';
-- REAL coupled constraint in the declarative schema, exactly like sandbox (B-008);
-- the migration performs the table rebuild a cross-column CHECK requires on SQLite:
CHECK ((worktree_mode = 'ref') = (worktree_ref <> ''))
```

**Side-table-vs-JSON decisions (all matchable state in columns; JSON blobs forbidden):**

- Session/run **binding** = a single scalar per row → **column** (`worktree_id`), not a side table and never `metadata_json` (L-003; filtered/joined by SQL in list scoping).
- **Git status** = **side table** `worktree_status`: different write cadence than identity, all-nullable unknown semantics, replaced wholesale per refresh.
- **Forge status** = **side table** `worktree_forge_status`: network-derived, provider-attributed, independently invalidated; keying through the worktree FK makes cross-workspace leakage structurally impossible.
- **Profile policy** = **flat columns + CHECK** on `task_execution_profiles`, byte-for-byte the sandbox precedent (`71_task_profiles.sql:19-39`).
- Discovered-but-unadopted worktrees = **no storage at all**; derived live from `git worktree list --porcelain -z` and merged into read payloads.

### Key flows

**Create (manual)**: validate git capability + git-backed workspace → derive name (given or generated adjective-noun, ADR-011) → sanitize branch/dir preview → refuse collisions (`UNIQUE(workspace_id,name)`, live-path index, branch-held checks below) → dispatch `worktree.pre_create` (sync-eligible: an explicit hook deny refuses with `worktree_denied_by_hook` + the hook's reason; a hook execution error fails open) → insert `pending` row → under the per-repo lock: `git branch <branch> <base>` (new branch; base = given ref or repo default) or reuse existing branch, `git worktree add <path> <branch>` (record `created_head`, `base_ref`, write `git config branch.<branch>.gh-merge-base <base>` for created branches) → bootstrap (copy list → setup command) → `ready` (+ `setup_state`) → dispatch `worktree.created`. Creation is a **durable phased state machine** (B-003): each step records `pending_phase` (`branch → checkout → copy → setup`) before executing, so ownership checkpoints (minted branch, added checkout) are always known. `CancelCreate` (surfaced as a cancel endpoint and the composer's cancel gesture) is valid only while `pending`: it rolls back recorded artifacts in reverse and deletes the row — canceled pending rows carry no history value. Any git/copy failure unwinds the same way (worktree remove --force if the checkout landed, compare-safe `update-ref -d` for the minted branch, delete the row) and surfaces the cause — retrying the same name works (US-001 EC-3). Per-run materialization is the same phased primitive invoked through `MaterializeForRun` — there is no second creation path (L-004). Branch-held refusals name the holder: existing worktree (`branch_held_by_worktree` + holder ref, US-002 EC-1) or the root checkout (`branch_checked_out_at_root`, EC-2).

**Adopt**: validation runs under the repository lock and proves **registered linked-worktree identity**, not just shared ancestry (B-006): (1) read `.git` (file or dir) without mutation, resolve the git dir and common dir (read `commondir`; fallback `rev-parse --git-common-dir`), canonical-compare against the workspace repo's common dir; (2) require the candidate to appear in a strictly parsed `git worktree list --porcelain -z` with an **exact canonical path match**, and require the admin `gitdir` linkage to hold in both directions (the path's `.git` points into `<common-dir>/worktrees/<name>`, and that admin dir's `gitdir` file points back at the path). Rejected classes, each with its named refusal: main checkout (`adoption_main_checkout`), foreign repo (`adoption_foreign_repository`), unreadable (`adoption_unreadable`), and malformed/ambiguous/bare/submodule/`core.worktree`-redirected/symlink-escaping candidates (`adoption_unreadable` with the classified cause) — the directory is never modified. Success mints the row (`origin=adopted`, `state=ready`, `git_dir` recorded as the immutable identity fingerprint, name = branch-derived label with collision suffix), no bootstrap (US-009 AC-2), dispatch `worktree.adopted`. Idempotent: adopting an already-adopted path returns the existing row (US-009 EC-1). Creation-side path rule (same blocker): an explicit `path` override must resolve inside approved roots — the configured `worktrees.root` or an operator-passed absolute path that overlaps **no** workspace root, no existing worktree, and no ancestor/descendant of either; containment is re-validated (canonicalized) immediately before every copy, setup, git, or filesystem mutation.

**List/discovery merge**: DB rows (states ≠ dismissed) + `git worktree list --porcelain -z` (cached per workspace, TTL `[worktrees].discovery_cache_ttl`, `refresh=true` bypasses) → entries not in the registry and not the main checkout render as `discovered` (prunable ⇒ marked stale, non-selectable); rows whose path vanished from both git and disk flip to `missing` (event `worktree.missing`; never a delete — ADR-006/007). Discovery never mutates git state; no implicit fetch/prune. The list payload carries per-workspace `repo` metadata (`git_backed`, `git_available`, diagnostic code) so clients gate affordances truthfully.

**Status**: single `git status --porcelain=v2 --branch -z` + two `git diff --numstat -z` passes (staged+unstaged) → branch/detached/dirty counts/±; ahead/behind from `# branch.ab` when upstream exists, else NULL (absent, not zero); no-upstream ahead may derive from `rev-list --count <base>..HEAD` labeled accordingly. Failures write `read_error` (explicit error state; blocks exit actions, US-019 AC-3). Refresh triggers: explicit gesture, list refresh, exit-action completion, bound-session turn end. Forge half: only on explicit refresh or after exit actions (no unrequested network, BR-14), via `ForgeProvider.Status`, cached in `worktree_forge_status` with `fetched_at` staleness.

**Exit plan** (computed server-side, one source of truth for UI/CLI/tools): from status + forge cache + bound-session activity → ladder position (`commit → commit&push → push → open_pr → view_pr`), per-action `{enabled, blocked_reason}` with the exact copy contract from the design set; global pauses: bound session running (`worktree_session_active`), status read failed (`worktree_status_unreadable`); diverged/behind blocks with no auto-resolution (BR-25). Cleanup evidence block: forge `merged` (fresher-wins vs local, BR-28) and local evidence (branch holds no unique commits / exists on remote). Forge-facing labels in the plan (request noun, action labels) come from the serving extension's `forge/capabilities` vocabulary — core hardcodes no forge terminology; the browser compare link comes from the extension's `compare_url_template` when one serves the remote, else from the core's minimal URL-shape table for well-known public hosts (github.com, gitlab.com, bitbucket.org), else the remote's web URL (ADR-010 — keeps US-022 AC-4 true with zero extensions).

**Exit actions**: `RunExitAction{action: commit|commit_push|push|open_pr, message?, title?, body?, draft?, base?}` mints a durable `worktree_exit_ops` row — the one-active-op-per-worktree rule is the partial unique index, not process memory, so it survives restart (B-005); detached from the request (`context.WithoutCancel`, SD-010) with `CancelExitAction(workspace, worktree, op_id)` as the explicit **compare-and-cancel** (a stale cancel for a finished op is a deterministic no-op and can never hit a later op); exit subprocesses run under process-group supervision (kill + wait on cancel/timeout); at boot, `running` ops are marked `failed` (interrupted) — completed steps stand, the report says where it stopped; steps emit durable progress events replayed over the per-worktree stream; per-step outcomes include skip reasons ("push skipped — already up to date") and exactly one next-step CTA on success. Commit = stage-all + commit with the scope shown honestly beforehand: counts + ± **and the untracked additions listed by name** (bounded, expandable — the risk class an operator must see before confirming; B-011); gitignored files are never staged (`add -A` respects the ignore rules — `.gitignore` remains the exclusion mechanism); the anti-pattern ADR-004 rejected was *silent, unlisted* staging, not breadth. Empty message ⇒ default `Update N files` (no daemon model-call generation in v1; the agent-staged path US-025 covers "write it for me"). Push publishes with `-u <remote> <branch>` when no upstream. `open_pr` resolves base by the recorded chain (`branch.<b>.gh-merge-base` → upstream → repo default), reads an unambiguous PR template from the base tree as body prefill (abstains when multiple, US-022 EC-1), calls `ForgeProvider.CreatePR` (idempotent: `opened_existing` surfaces the PR), or returns the browser compare URL in the zero-credential tier. Mid-chain failure attributes to its step; completed steps stand (US-023 EC-1/EC-2).

**Remove**: the flow opens with an atomic CAS `ready → removing` (B-004) — the fence that makes new session bindings, policy resolutions, and exit actions refuse (`removing` ≠ ready, invariant 2) while removal is in flight; any refusal or failure transitions back to `ready`. Lock order is fixed: DB state fence first, repository lock second. Then: refresh safety evaluation — evaluation failure blocks (`worktree_safety_check_failed`; a read error never counts as clean, BR-31). Bound sessions must stop first (the flow stops them; a still-running session blocks with the reason). After the refusal checks pass, dispatch `worktree.pre_remove` (sync-eligible; payload carries `{worktree, force, risk}`; explicit deny blocks with `worktree_denied_by_hook`, hook execution errors fail open). Under the repository lock, safety evidence, branch head, recorded `git_dir` identity, and session activity are **re-read immediately before the destructive mutation** — the pre-lock evaluation is advisory, the under-lock one is authoritative (TOCTOU closure). Clean → `git worktree remove <path>`; dirty or unpushed-unique → structured refusal (`worktree_dirty_requires_force` / `worktree_unpushed_requires_force`) carrying the risk inventory (changed-file count, ±, unpushed commit count); unique-commits-on-remote downgrades to informational (US-027 EC-1). `force=true` (separate explicit argument) runs `git worktree remove --force`. Leftover directory after a forced remove is deleted only after verifying its `.git` `gitdir:` pointer resolves under the repo's `<common-dir>/worktrees/` (herdr recovery, BR-35). Branch never deleted; run-created branches reclaim via `git update-ref -d refs/heads/<b> <created_head>` only when `created_branch=1` ∧ namespace-recorded ∧ pointer unchanged (ADR-011). Row → `removed` tombstone; `worktree.removed` dispatched; history remains readable and named (US-026 AC-3).

**Reconcile & boot recovery**: `RecoverCreations` at daemon boot resolves stale `pending` rows from their **recorded phase**, never from directory existence (B-003): phase `branch`/`checkout` incomplete ⇒ unwind recorded artifacts ⇒ `failed` with retry; phase `copy`/`setup` (checkout durably complete) ⇒ `ready` with `setup_state=failed` + `setup_error="interrupted by daemon restart"` (truthful, usable-but-flagged; bootstrap is not silently re-run). Stale `removing` rows re-run the removal decision; stale `running` exit ops are marked `failed` (interrupted). Reconciliation flips vanished worktrees to `missing` only; a restore requires the directory's admin gitdir to match the recorded `git_dir` fingerprint — a different repo at the old path stays `missing` (US-028 EC-1/EC-2). `Dismiss` moves `missing`/`removed`/`failed` rows to `dismissed` (hidden from nav; deletes nothing else).

**Bootstrap contract**: (1) copy list — patterns from `[worktrees].copy_list` (global + workspace overlay merge); candidate set = `git ls-files --others --ignored --exclude-standard -z -- <patterns>` in the source checkout (ignored files only by construction, BR-36); copy with no-overwrite semantics and a nesting guard. (2) setup command — `[worktrees].setup_command` (typically the workspace overlay; operator-authored config, never agent input) runs in the new worktree via the operator's shell with a **minimal allowlisted environment** (`PATH`, `HOME`, `SHELL`, `LANG`/`LC_*`, `TMPDIR` + the `COMPOZY_WORKSPACE_ROOT`, `COMPOZY_WORKTREE_PATH`, `COMPOZY_WORKTREE_ID`, `COMPOZY_WORKTREE_BRANCH` contract vars — no daemon secrets, no `GIT_*`), bounded by `setup_timeout` with **process-group kill + wait** on expiry; captured output is size-capped and passes the shared redaction boundary before persisting into `setup_error` or any event (B-007); failure ⇒ `setup_state=failed` + readable `setup_error`, worktree stays usable-but-flagged (US-029 AC-3). (3) `worktree.created` hook event fires as the same lifecycle surface (US-033 AC-2), fail-open. Adoption runs none of this by default.

**Session binding**: `CreateOpts.Worktree` resolves via `SessionWorktreeResolver` → binds `sessions.worktree_id` and sets `CWD` to the worktree root. Containment (all four gates) validates the requested cwd against the **bound worktree's root** when bound, else the workspace root — registry-evaluated, never path-prefix heuristics (ADR-005). Resume re-resolves the binding; a missing/removed worktree fails resume with `worktree_missing` (never silently falls back to root). Sandbox: `LocalRootDir` = the bound worktree root; the resolved parent workspace continues to own config/agents/skills/memory (memory store keeps the parent `workspaceRoot` — verified directory-scoped). New-session composer "new worktree" = create (pending visible via stream) then session create bound to it; cancel rolls the creation back. Fork (`/worktree` + menu on live sessions) = confirmation → fresh clean session bound to the target worktree (grill decision); origin session untouched. **Child-session spawn inherits the binding structurally** (B-014): the canonical spawn request gains an internal, non-user-selectable inherited `worktree_id` — a child of a worktree-bound parent resolves through the same `SessionWorktreeResolver`, is contained by the same worktree root, and a missing/not-ready binding refuses the spawn exactly like session create; there is no path for a child to fall back to the workspace root or select a different checkout in v1 (choice of a different child worktree is the deferred part, not inheritance safety).

**Task policy application** (B-002 — one enforceable contract): the **enqueue transaction resolves the profile once** (`inherit` ⇒ config default ⇒ `none` semantics) and writes the normalized outcome onto the run row (`task_runs.resolved_worktree_mode/_ref` — the same snapshot discipline as `resolved_network_participation`). The session bridge consumes **only the claimed run's snapshot**, never the live profile, so a post-enqueue profile edit can never retarget an enqueued run. Resolution at worker start: `none`/empty ⇒ root; `ref` ⇒ re-resolve the snapshotted ref (fail `worktree_ref_invalid`); `per_run` ⇒ `MaterializeForRun` (thin wrapper over the phased creation primitive; name = task-slug + run-suffix; branch `<namespace><task-slug>-<run-suffix>`; same `worktree.pre_create` gate — a hook deny fails the run with `worktree_denied_by_hook`; bootstrap; failure fails the run and unwinds — no orphan, BR-21). Materialization runs **inside the active lease**: the claim heartbeat continues throughout (bootstrap ≤ `setup_timeout` < lease semantics stay owned by `internal/task`), every run/session binding and run-fail/terminal write goes through `task.Service` guarded by the claim token, and a claimant whose lease lapsed mid-materialization cannot bind — the worktree unwinds via the standard creation rollback. The worktree domain never receives the claim token (B-007); it sees task/run identifiers only. `taskRoleProfileFingerprint` gains worktree mode+resolved ref; `per_run` sessions are never reused (BR-23 structural). Fan-out isolation = the same `per_run` semantics per spawned run with per-run failure attribution (US-016). Coordinator: never bound (ADR-003); `HealthySession` treats worktree-bound workers as same-workspace; `PromptOverlay` names each worker's worktree (awareness).

**Loop resolution**: at action bind, `EnvironmentSpec` resolves exactly like the task policy (`worktree` ⇒ re-resolve ref; `per_run` ⇒ materialize per node execution instance — under fan-out, per branch instance, US-018 EC-3; `directory` ⇒ template-render then contain against workspace root; unresolvable ⇒ node fails before any session starts). Loop-level default lives on `LoopConfig` (configure dialog + per-run override); node setting wins. Linter gains reason codes for invalid environment specs.

### API Endpoints

All routes registered identically in `internal/api/httpapi` and `internal/api/udsapi` (shared `BaseHandlers`); contract types in `internal/api/contract/worktrees.go`; OpenAPI + TS regenerated by `make codegen`.

| Method + path | Purpose | Notes |
|---|---|---|
| `GET /api/workspaces/:workspace_id/worktrees` | List (registry + discovery merge) | `?refresh=true` forces git scan; payload: worktrees[], discovered[], repo{git_backed, git_available, diagnostic} |
| `POST /api/workspaces/:workspace_id/worktrees` | Create | `{name?, branch?, base_ref?, existing_branch?, path?}`; 202 + pending payload; progress via streams |
| `POST /api/workspaces/:workspace_id/worktrees/adopt` | Adopt by path | `{path}`; validation refusals below; idempotent |
| `GET /api/workspaces/:workspace_id/worktrees/:worktree_id` | Inspect | Record + last status + forge cache + bindings summary |
| `GET .../worktrees/:worktree_id/status` | Status | `?refresh=true`; `?forge=true` explicitly refreshes forge tier |
| `GET .../worktrees/:worktree_id/exit` | Exit plan | Ladder + per-action blocked reasons + cleanup evidence |
| `POST .../worktrees/:worktree_id/cancel` | Cancel a pending creation | Pending only (`worktree_not_pending` otherwise); rolls back artifacts, deletes the row (B-003) |
| `POST .../worktrees/:worktree_id/exit/actions` | Run exit action | `{action, message?, title?, body?, draft?, base?}`; 202 `{op_id}` (durable `worktree_exit_ops` row); detached lifetime |
| `POST .../worktrees/:worktree_id/exit/cancel` | Cancel exit op | `{op_id}` required — compare-and-cancel; stale/finished op ⇒ deterministic no-op (B-005) |
| `PATCH /api/tasks/:id/execution-profile/worktree` (new) | Set worktree policy alone | Read-modify-write primitive preserving every other profile block (US-014 AC-3, B-010); same active-run lock as PUT |
| `DELETE .../worktrees/:worktree_id` | Remove | `?force=true` second step; 409 + machine-readable refusal payload first |
| `POST .../worktrees/:worktree_id/dismiss` | Dismiss tombstone | History preserved |
| `GET .../worktrees/:worktree_id/stream` | Per-worktree SSE | Durable events, `after_sequence` replay (exit progress, state flips) |
| `GET /api/worktrees/catalog-stream` | Global catalog SSE | Named event `worktree_catalog_changed {kind, workspace_id, worktree_id}`; operator-surface only (same trust tier as the session catalog stream), frames workspace-attributed for workspace-qualified cache invalidation — never consumed by agent tools (B-001) |
| `POST /api/workspaces/:workspace_id/sessions` (existing) | + `worktree` ref or `new_worktree{name?}` | Additive contract change |
| `GET /api/sessions` (existing) | + `worktree` filter param | Additive |
| Task profile routes (existing) | + `worktree` policy block | PUT-replace semantics unchanged |

**Deterministic error codes** (one per cause; identical across HTTP/UDS/CLI/tools — BR-39):

`worktree_git_unavailable` · `worktree_git_version_unsupported` · `workspace_not_git_backed` · `worktree_name_taken` · `worktree_path_exists` · `branch_held_by_worktree` (payload names the holder) · `branch_checked_out_at_root` · `base_ref_not_found` · `repo_has_no_commits` · `worktree_not_found` · `worktree_not_ready` · `worktree_pending` · `worktree_missing` · `worktree_ref_invalid` · `adoption_main_checkout` · `adoption_foreign_repository` · `adoption_unreadable` · `worktree_operation_in_progress` · `worktree_session_active` · `worktree_status_unreadable` · `worktree_dirty_requires_force` · `worktree_unpushed_requires_force` · `worktree_safety_check_failed` · `worktree_removal_failed` · `per_run_materialization_failed` · `worktree_config_invalid` (names the offending key, US-030 EC-1) · `worktree_denied_by_hook` (carries the denying hook's name + reason) · `worktree_not_pending` (create-cancel on a non-pending row) · `forge_unavailable` · `forge_error` (with provider cause).

## Integration Points

- **System git** (≥ 2.37): capability-probed at service init and per-list; degradation is a diagnostic, never a raw error (PRD constraint). Commands run under the operator's git config and credential helpers (push auth = user's ssh-agent/credential manager).
- **`forge.provider` extensions** (ADR-010): daemon→extension service methods `forge/capabilities`, `forge/status`, `forge/pr_create`. `forge/capabilities` is the forge-agnosticism contract: per served remote it returns the provider vocabulary (request noun + action labels), `supports_draft`, `compare_url_template`, template path conventions, and `credential_source` — core/web render from it, so GitLab and peers are pure extensions. Errors carry machine-readable causes (`rate_limited`, `credential_expired`, `unsupported_remote`); no retry storms — one attempt per user gesture, staleness marked.
- **Bundled GitHub extension** (`extensions/forge/github`): manifest `capabilities.provides=["forge.provider"]`, `requires_env=["GITHUB_TOKEN"]` (secret binding), subprocess self-hosted via `compozy __internal extension-provider forge-github`; serves github.com only in v1 (custom/enterprise hosts are a post-v1 extension-side increment — zero core change); credential ladder: binding → `gh auth token` (reported as `credential_source:"gh"`) → absent (affordances absent). Outbound HTTP uses explicit-timeout clients (no `http.DefaultClient`).
- **Marketplace GitHub remote MCP**: unchanged and unreferenced by this feature (agent tools live there; ADR-010 keeps the surfaces independent).

## Impact Analysis

| Component | Impact | Description / risk | Action |
|---|---|---|---|
| `internal/session` containment | modified (security-shaped) | Boundary widens to bound attached worktree roots; 4 gates must agree | One resolution helper + invariant tests per gate |
| `internal/workspace` prune-on-list | untouched, quarantined | Worktree reconciler must never route through `Unregister` | Guard + regression test |
| `sessions`/`task_runs` tables | modified | New nullable FK columns on hot tables | Single migration; index-backed list filters |
| `task_execution_profiles` | modified | Two columns + CHECK; PUT-replace footgun inherited | Tool description warning (existing pattern) |
| Loop DSL (`compozy.loop/v1`) | modified (contract-bearing) | `cwd` → `Environment`; session binding key input changes | Alias normalization + linter + binding tests |
| Native tool catalog | modified | New toolset + 4 tools + input changes on 6 existing tools (`additionalProperties:false` ⇒ contract change each) | `eng-contract-codegen-coship` loop, digest updates |
| Extension protocol | modified | 7th provide surface + 3 service methods | Conformance matrix + docs |
| Web workspace/session/tasks/loops/os systems | modified | Per `_uiux.md`; store persist key v2→v3 | Follow the component plan verbatim |
| OpenAPI / generated TS | modified | New routes + payloads | `make codegen` in the same change |

## Testing Approach

- **Unit**: fake `GitRunner` (scripted stdout/stderr/exit), table-driven porcelain parsers, naming/slugging, exit-plan computation, policy normalization/validation, error mapping. No real git.
- **Integration** (`+integration` tag): real git in `t.TempDir()` repos — create/adopt/remove/reclaim/status against actual worktrees, per-repo lock contention, bootstrap copy/setup, reconcile flows, migration fresh/reopen/integrity suites.
- **E2E runtime** (Go harness + acpmock): session-in-worktree lifecycle, task per-run materialization, containment enforcement, API/CLI/tool parity, forge via a stub extension.
- **E2E web** (Playwright): nested nav, create/adopt/remove dialogs, composer materialization, task setup policy, exit ladder — per `_uiux.md` states.
- Verification commands per phase: `make gate` during tasks; `make test-integration`, `make test-e2e-runtime`, `make test-e2e-web` on the phases that own them; `make gate-full` at workstream close. **`make codegen-check` and the docs/site build are explicit acceptance gates for every contract-bearing task** — generated OpenAPI/TS/CLI/API references co-ship or the task is incomplete (B-015). Every concrete case with IDs lives in `_tests.md`; at task-generation time each case is annotated with its invariant, owning layer, and canonical suite (test-placement rule) — cases extend existing canonical suites, never one suite per ID.

## Development Sequencing

### Build Order

1. **Schema + store** (C2): tables, binding columns, profile columns, sqlc, migration, fresh/reopen/integrity suites.
2. **Worktree core** (C1a): entity/errors/store, git runner + capability gate, porcelain parsers, naming, per-repo lock.
3. **Lifecycle** (C1b): create + bootstrap + rollback; adopt + validation; discovery/list merge; status.
4. **Safety** (C1c): remove + refusals + leftover recovery + reclamation; reconcile + boot recovery; dismiss.
5. **Events** (C7): hook family + canonical events + coverage matrix rows.
6. **Session binding** (C3): CreateOpts, containment gates, resume validation, sandbox root, fork surface, composer/new-session contract.
7. **API + streams** (C8): contract, handlers, HTTP/UDS routes, per-worktree stream, catalog stream, OpenAPI.
8. **CLI** (C9) and **native tools** (C10).
9. **Task policy** (C4+C5): 11 layers + bridge + fingerprint + fan-out.
10. **Loop environment** (C6): DSL + resolution + linter + web inspector descriptor.
11. **Exit engine** (C1d): plan + actions + progress; browser-tier PR path.
12. **Forge surface** (C12+C13): protocol + registry + conformance + bundled GitHub extension.
13. **Web** (C14): data layer → nav (S1–S3, S5) → create/adopt/remove (S4, S15) → session (S7–S9, S16) → task/loop (S10–S13) → exit (S6, S14).
14. **Docs** (C15) + config examples + official `skills/compozy/` update.

### Technical Dependencies

Steps 1–2 block everything; 6 blocks 9–11; 7 blocks 8, 13; 11 blocks 12's exit integration (browser tier ships in 11, forge tier in 12). No external/team dependencies; system git present on dev/CI machines (already required by `gitsrc` floors).

## Monitoring and Observability

- Canonical events (registry entries, `internal/events/registry_worktree.go`) — this list is exhaustive; anything a test or stream consumes appears here (N-003): `worktree.created` · `worktree.adopted` · `worktree.removed` · `worktree.missing` · `worktree.dismissed` · `worktree.creation_canceled` · `worktree.setup_failed` · `worktree.status_refreshed` · `worktree.branch_reclaimed` · `worktree.exit_action_started` · `worktree.exit_action_step` · `worktree.exit_action_completed` · `worktree.exit_action_failed` · `worktree.exit_action_canceled`. SSE frame names: per-worktree stream re-emits the canonical names; the catalog stream emits `worktree_catalog_changed`. Live broadcasters publish only after durable append; streams replay by `after_sequence`; all diagnostic strings pass the shared redaction boundary before append (B-007).
- `worktree_id` joins the correlation-key set (alongside `workspace_id`, `session_id`, `run_id`) in every worktree-scoped event and log line.
- Hook events: `worktree.pre_create` and `worktree.pre_remove` are **sync-eligible** — an explicit deny verdict blocks the operation with `worktree_denied_by_hook` (hook name + reason in the payload), a hook execution error fails open; `worktree.created` / `worktree.adopted` / `worktree.removed` are observe-only and fail-open, with `{worktree_id, workspace_id, name, branch, path, origin}` payloads — one event per worktree per transition, ordered per worktree (US-033). `pre_create` gates manual and per-run creation alike (a denial fails the run deterministically); adoption is ungated in v1 (non-destructive, reversible via dismiss).
- Coverage matrix test extends to fail if any worktree lifecycle path stops emitting its canonical event.

## Safety Invariants

1. Session cwd containment is registry-evaluated: an unbound session is contained by its workspace root; a bound session by its bound worktree's root; nothing else — and all four gates (`ResolveSessionCWD`, `sandboxRuntimeCWD` ×2, sandbox tool-host root) resolve from the same binding. Path-prefix heuristics are forbidden.
2. Only attached (`ready`) worktrees grant session eligibility; `pending`/`failed`/`missing`/`removed` bindings fail deterministically, never fall back to the root (BR-19).
3. Every git mutation for one repository serializes under the per-repo lock keyed by canonical common dir — a plain in-process mutex with bounded fail-fast waiters (overflow refuses with `worktree_operation_in_progress`). It owns no durable work, no retry, no claim, no lease, and no terminalization; `task_runs` remains the only queue (L-003).
4. At most one exit action runs per worktree, enforced by the durable `worktree_exit_ops` partial unique index (survives restart); exit execution detaches from request lifetime (`context.WithoutCancel`, SD-010), cancellation is compare-and-cancel on `(workspace, worktree, op_id)`, and boot marks interrupted `running` ops `failed`.
5. Exit actions pause while any bound session is executing; the pause reason is stated on every surface (BR-24).
6. A status read error is an explicit blocking state — no dependent action ever runs on unknown/stale-presented-as-clean data (BR-13, BR-31).
7. Session-reuse fingerprints include the worktree axis; `per_run` sessions are never reused — a session can never execute in a worktree other than its binding (BR-23).
8. Per-run materialization failure fails the run and unwinds completely: no directory, no branch, no row (BR-21).
9. Worktree reconciliation never deletes rows or dependent records; out-of-band removal produces `missing` tombstones only, and the workspace prune-on-list path is never applied to worktrees (ADR-006/007).
10. Worktree removal never deletes branches; reclamation only on `created_branch=1` + recorded namespace + `update-ref -d` compare-and-delete against `created_head` (ADR-011).
11. Leftover-directory cleanup after a forced removal verifies the `gitdir:` pointer resolves under the repo's `<common-dir>/worktrees/` before touching anything (BR-35).
12. Adoption validates repository identity before minting anything and never modifies a refused directory (BR-10).
13. Discovery and status reads never mutate git state; no implicit fetch or prune ever runs (BR-8, BR-14).
14. Worktree identity is minted by the runtime and never derived from files inside the tree; the parent's identity file inside a checkout is never read as the worktree's (ADR-001).
15. Create/adopt are idempotent under retry: the same intent yields the prior result or a named collision — never two worktrees (US-007 EC-3, US-009 EC-1).
16. Forge credentials (binding or `gh`-sourced) exist only inside the extension process; no token transits logs, payloads, events, SSE, or memory (Security Invariants).
17. Worktree policy is decided before enqueue and **snapshotted onto the run row in the enqueue transaction**; the session bridge consumes only the claimed run's snapshot, materialization stays inside the active lease with heartbeats, every binding/terminal write goes through `task.Service` under the claim token, and a stale claimant can never bind a worktree or start a session — claim authority, lease semantics, and terminal states are untouched (L-003/004/005).
18. Every service read/mutation and stream subscription is predicated on `(workspace_id, worktree_id)` — enforced by the store, the composite UNIQUE target, and composite binding FKs, not by handler convention; a cross-workspace reference returns `worktree_not_found` with no existence leak, and no surface may list, stream, or cache a worktree across workspaces (BR-7).
19. Only an explicit `worktree.pre_create`/`worktree.pre_remove` deny verdict blocks an operation (with hook name + reason); hook execution errors and observe-event consumer failures never block a lifecycle transition (fail-open).
20. Child sessions spawned from a worktree-bound parent inherit the binding through the same resolver and containment; a missing/not-ready inherited binding refuses the spawn — no child ever silently falls back to the workspace root (B-014).

## Extensibility Integration Plan

- **Provide surfaces**: `forge.provider` added to the closed set (`internal/extensionprotocol/capabilities.go`) with service methods `forge/capabilities`, `forge/status`, `forge/pr_create`; consumer registry + manager wiring + conformance-matrix coverage; protocol docs in the extension docs section.
- **Bundled extensions**: `extensions/forge/github` (new) via the `dev-cycle` managed-install pattern.
- **Hooks**: new `worktree` family — `worktree.pre_create` + `worktree.pre_remove` (sync-eligible: extensions/hooks can deny with a reason — policy gating such as "no worktrees outside deploy hours" or "protect this branch's worktree") and `worktree.created`, `worktree.adopted`, `worktree.removed` (observe-only; payload above); introspection descriptors + matcher support (`worktree_id` joins matchable keys; `workspace_root` matching already covers worktree roots via payload path). Extension provisioning reacts to `worktree.created` fail-open; its outcomes live in the extension's own logs/records — `setup_state` covers only the config-owned copy list + setup command (grill decision; a first-class `worktree.provisioner` provide surface is a post-v1 additive increment).
- **Host API**: **no new methods** — forge flows are daemon→extension; the extension needs no daemon callbacks for v1 (checked against the 95-method catalog; nothing added, nothing changed).
- **MCP**: no change — agent-facing GitHub tools remain the marketplace remote MCP entry (ADR-010).
- **Registries/catalogs**: native tool catalog + toolset registry gain the worktree entries; marketplace surfaces unchanged.
- **Bridge SDKs**: no impact — bridges deliver task notifications and carry no execution-environment semantics (checked `internal/bridgesdk` contract types; no worktree datum crosses them).
- **Official Compozy skill** (`skills/compozy/`): updated in the same program — worktree verbs, native tools, policy modes, exit flow, and error codes documented for agents.

## Agent Manageability Plan

- **CLI** (`compozy worktree …`, all with `-o json|jsonl` and deterministic exit codes): `list [--refresh]` · `create [name] [--branch] [--base] [--existing-branch] [--path]` · `cancel <ref>` (pending creation) · `adopt <path>` · `inspect <ref>` · `status <ref> [--refresh] [--forge]` · `exit <ref>` (plan) · `commit <ref> [-m] [--push]` · `push <ref>` · `pr <ref> [--title] [--body] [--draft] [--base]` · `exit-cancel <ref> --op <op_id>` · `remove <ref> [--force]` · `dismiss <ref>`. Worktree policy alone: `compozy task profile set-worktree <task> --mode <inherit|none|ref|per_run> [--ref <worktree>]` (the patch primitive — other profile blocks untouched, B-010). Session start: `compozy session new --worktree <ref> | --new-worktree [name]` (mutually exclusive with `--cwd`, whose auto-register meaning is unchanged and documented against the new flags). Task policy rides `compozy task profile update`. Workspace resolution uses the existing ladder; agent identity via `COMPOZY_SESSION_ID`/`COMPOZY_AGENT` with the same cross-workspace enforcement.
- **HTTP/UDS**: full parity via shared `BaseHandlers` (route table above); structured payloads identical across transports and equal to CLI JSON (US-031 AC-1).
- **Native tools**: toolset `compozy__worktree` — `compozy__worktree_list` (RiskRead) · `compozy__worktree_inspect` (RiskRead) · `compozy__worktree_create` (RiskMutating) · `compozy__worktree_remove` (RiskDestructive, `Destructive: true`; dirty two-step identical to every surface). Targeting rides the owning tools: `compozy__session_create.worktree`, `compozy__task_execution_profile_set.worktree` (whole-profile PUT-replace, caveat documented), **`compozy__task_worktree_policy_set`** (the patch primitive — sets the worktree block alone, preserving every other block; the canonical agent path, B-010), `compozy__task_fanout_runs.worktree_per_run`, loop create/configure environment fields. All mutations flow the session permission mode; removal always classifies destructive (BR-40).
- **Discovery**: `worktree list`/`inspect` expose the full state vocabulary exactly as the UI renders it (US-031 EC-1); config discoverable via existing `compozy config` surfaces; git capability diagnostics ride list/inspect payloads.
- **Deterministic errors**: the single error-code table above, byte-identical across CLI/HTTP/UDS/tools.

## Config Lifecycle

| Key | Default | Purpose |
|---|---|---|
| `worktrees.root` | `""` ⇒ `$COMPOZY_HOME/worktrees` | Placement root; layout `<root>/<workspace-name>/<worktree-name>` (ADR-005) |
| `worktrees.run_branch_namespace` | `"run/"` | Namespace for per-run branches; eligibility recorded per branch at creation (ADR-011) |
| `worktrees.copy_list` | `[]` | gitignore-syntax patterns; ignored files only (BR-36) |
| `worktrees.setup_command` | `""` | Post-create provisioning command (typically set per workspace) |
| `worktrees.setup_timeout` | `"10m"` | Bound on the setup command (BR-37) |
| `worktrees.discovery_cache_ttl` | `"30s"` | Git-scan cache TTL for list merging |
| `task.orchestration.profile.default_worktree_mode` | `"inherit"` | Profile default (ADR-009) |

- **Structs/defaults/merge**: `WorktreesConfig` in the `Config` root; defaults in `internal/config/defaults`; `worktreesOverlay` + `Apply` in the merge overlay; per-workspace override via the existing `<root>/.compozy/config.toml` overlay (US-030 AC-2) — worktree config is read from the **parent** workspace's overlay chain, never from a worktree's own tree.
- **Validation**: `root` absolute-or-empty and writable at use (invalid ⇒ `worktree_config_invalid` naming the key, US-030 EC-1); namespace matches `^[a-z0-9._-]+(/[a-z0-9._-]+)*/$`; durations parseable and positive; patterns non-empty.
- **Lifecycle matrix**: `worktrees.*` ⇒ `Live` (affects future creations/refreshes only; nothing moves on disk, BR-38); `task.orchestration.profile.default_worktree_mode` ⇒ `Live`.
- **Docs/examples/tests**: `config-toml.mdx`, `lifecycle-matrix.mdx`, example `config.toml`s, and config validation tests all move in the same change (SD-011). Named co-ship owners and gates (N-005): the `internal/config` struct/defaults/overlay/validation/lifecycle-matrix files, `internal/settings` sections projection, generated CLI reference + generated API reference + `openapi/compozy.json` → `web/src/generated/compozy-openapi.d.ts`, the `packages/site` build, and the `skills/compozy/` runtime-truth audit — verified by `make codegen-check` and the site build in the same change.

## Web/Docs Impact

- **Web**: `_uiux.md` §Component plan is the binding contract — surfaces S1–S16, one new `@compozy/ui` primitive (`SplitButton`, with story+test), worktree domain components/data layer under `web/src/systems/workspace/` plus session/tasks/loops/os insertions at the verified points; `active-workspace-store` persist key bumps `compozy:active-workspace:v2` → `v3` with a worktree context field; sessions/tasks list filters gain the `worktree` param; new catalog + per-worktree streams wire the two-tier invalidation pattern. Visual contract: `docs/design/opendesign/worktree/` (16 artboards + `DESIGN-NOTES.md` locked decisions).
- **Docs (`packages/site`)**: new "Worktrees" section — index/concepts, creating & adopting, navigation & selection, task isolation (per-run + fan-out), loop environments, assisted exit (+ GitHub extension setup: secret binding, `gh` fallback), removal & recovery, configuration, agent surface (CLI/API/tools) — plus generated CLI/API references, `config-toml.mdx`, extension docs for `forge.provider`, and the workspaces docs cross-links.
- **QA tracker**: this program adds `untested` content-addressed scenarios under `docs/qa/scenarios/` for each feature area (creation, adoption, navigation, session binding, per-run tasks, fan-out, loop env, exit ladder, PR flow, removal, missing recovery, config) — flagged at task time, walked in the program's QA tail per the standing workflow.

## Delete Targets / No-Compat Clauses

- Loop `params.cwd` is **deleted end to end** (B-009 hard cut): the DSL field, the web "Working dir" `FieldSpec` and its rendering path, and every fixture/doc mention go in one change; a definition still carrying `cwd` fails validation with a lint reason naming the one-line migration (`params.cwd` → `environment.directory`). No alias, no parse-time normalization, no legacy reader. US-018 EC-1 is amended in `_user_stories.md` in the same change.
- Web `compozy:active-workspace:v2` persisted state is **discarded** on the v3 key bump — no migration shim.
- No dual fields, no schema fallbacks, no legacy readers anywhere else: all storage, contracts, and payloads ship in their final shape in one change. There is no placeholder tier — surfaces without backing capability render absent, never disabled-forever.

## Technical Considerations

### Key Decisions

- **No daemon model-call text generation in v1**: commit/PR sheets use honest defaults ("Leave blank to use a default message"); "have the agent write it" stages a reviewable prompt (US-025). Satisfies BR-27 and the design set's generation-honesty rule without a new model-call surface. `_uiux.md` is reconciled to this truth (B-013): generation rows/states are removed from the S14 contract and the commit/PR artboards iterate accordingly — the UI renders only editable fields, honest defaults, and the agent-staged path.
- **State vocabulary mapping (N-004)**: the wire carries `failed` (lifecycle state) and `read_error` (status condition) as distinct facts; the UI `error` chip is the presentation of `read_error`, and `failed` renders as the failed lifecycle state — one wire vocabulary, one documented presentational mapping, never a merged `error` on the wire.
- **Copy list is config, not a dotfile**: no new repo-convention file; global + per-workspace overlay covers team sharing via the workspace's committed `.compozy/config.toml`. (synara's `.worktreeinclude` rejected as a second config system.)
- **Placement grouping by workspace name at creation time**: readable paths (`~/.compozy/worktrees/compozy/docs-refresh`); renames never move directories (labels come from the record, BR "display labels").
- **Discovered entries are unmaterialized**: no row until adoption — keeps the registry an intent log, and "select = adopt" (ADR-002) the only consent moment.
- **Exit plan computed daemon-side**: one truth source for the ladder + blocked reasons across web/CLI/tools; the web renders, never re-derives.
- **Run-created worktrees of a deleted task** survive as ordinary worktrees with standard evidence-based cleanup suggestions (PRD open question resolved — no special escalation; history rules already forbid cascades).

### Known Risks

- **Cross-platform path canonicalization** (case-insensitive FS, `/private/var`, Windows separators) touches lock keys, containment, adoption identity, and leftover verification — mitigated by a single canonicalization helper with per-platform tests; `GOOS=windows` cross-build gates subprocess work.
- **Bootstrap latency inside per-run starts** (up to `setup_timeout`) — accepted; pending state is visible, and the timeout is configurable.
- **Concurrent operator git usage** can tear status reads — reads classify failures as explicit `read_error`, never as clean (invariant 6).
- **Forge cache staleness/leakage** — staleness stamped (`fetched_at`), scoping structural via FK; extension errors degrade affordances with the cause named.
- **Contract fan-out** (6 native tools with `additionalProperties:false`, OpenAPI, CLI, TS) — each surfaced tool change follows `eng-contract-codegen-coship` in its owning task; digests updated together.

## Architecture Decision Records

- [ADR-001: Worktree as a Nested Sub-Context of Its Parent Workspace](adrs/adr-001.md) — product model + vocabulary.
- [ADR-002: Discovery Lists Every Git-Known Worktree; Adoption Is Selection With Identity Validation](adrs/adr-002.md) — discovery truth + adopt-on-select.
- [ADR-003: Creation Surfaces — Session, Tasks (Including Per-Run), and Loops; Automation Deferred](adrs/adr-003.md) — v1 surface set.
- [ADR-004: Assisted Exit With an Extensible Forge Layer](adrs/adr-004.md) — exit composition + forge tiering.
- [ADR-005: Central-Home Placement; the Session Boundary Extends to Attached Worktrees](adrs/adr-005.md) — placement + boundary extension.
- [ADR-006: Lifecycle Safety — Bootstrap Contract, Two-Step Removal, Branch Preservation, Reconcile-Never-Cascade](adrs/adr-006.md) — safety rules.
- [ADR-007: Worktree Storage — Global `worktrees` Table Plus Nullable Binding Columns](adrs/adr-007.md) — physical model.
- [ADR-008: Git Execution — A Capturing Runner Inside `internal/worktree`, Serialized Per Repository](adrs/adr-008.md) — git layer.
- [ADR-009: Task Worktree Policy — Profile Block Mirroring SandboxPolicy, Per-Run Materialization at the Session Bridge](adrs/adr-009.md) — task path.
- [ADR-010: Forge Provider — New `forge.provider` Extension Surface; Bundled GitHub Extension](adrs/adr-010.md) — forge surface + credentials.
- [ADR-011: Branch Namespace `run/`, Readable Generated Names, and Compare-and-Delete Reclamation](adrs/adr-011.md) — naming contract.

## Compozy Impact Audit

- **Native tools**: adds toolset `compozy__worktree` + 4 tool IDs (list/inspect/create/remove with risk classes above) + `compozy__task_worktree_policy_set` (RiskMutating patch primitive); changes input schemas + digests of `compozy__session_create`, `compozy__task_execution_profile_set`, `compozy__task_fanout_runs`, `compozy__loop_create`, `compozy__loop_configure`, `compozy__loop_run`; catalog snapshot + capability gates (git availability) updated together.
- **Extensibility and hooks**: `forge.provider` provide surface + 3 service methods + conformance; `worktree` hook family — five events: `pre_create`/`pre_remove` sync-eligible (explicit deny blocks with reason; execution errors fail open) dispatched at the create/materialize/remove call sites, and `created`/`adopted`/`removed` observe-only fail-open — never by tailing event tables; bundled `extensions/forge/github`; no Host API changes; MCP surfaces unchanged; bridge SDKs unchanged (checked — no execution-environment semantics cross bridges); config lifecycle per its section.
- **Workspace data isolation**: every new datum (`worktrees`, `worktree_status`, `worktree_forge_status`, binding columns) is workspace-scoped through `worktrees.workspace_id`; list/read/cache/SSE/event paths join through it (invariant 18); memory stays parent-workspace-scoped by construction (directory-keyed store + parent `workspace_id` in the catalog); tests assert cross-workspace list/stream isolation.
- **Official Compozy skill**: `skills/compozy/` updated with worktree verbs, tools, policy modes, exit flow, and error codes (build-order step 14).

## Assumptions / Defaults

1. Placement default `$COMPOZY_HOME/worktrees/<workspace-name>/<worktree-name>`; the canonical rendered form in docs/design is `~/.compozy/worktrees/<workspace>/<name>`.
2. Git floor 2.37 everywhere (shared with `gitsrc`'s documented floor); older git ⇒ the whole worktree surface degrades to diagnostics.
3. Generated manual names: embedded adjective+noun wordlists, `-N` suffix on collision; per-run names: `<task-slug>-<run-suffix>` with the suffix derived from the run id.
4. `git config branch.<branch>.gh-merge-base <base>` is written for every Compozy-created branch; it is the first link of the PR base-resolution chain.
5. Commit action stages the full working tree (`add -A`) with the scope shown before confirmation — counts + ± plus untracked additions listed by name (B-011); gitignored files never stage; staged-only/partial commits stay with the terminal in v1.
6. Exit progress rides durable events + per-worktree SSE; there is no polling contract.
7. Fork-to-worktree creates a fresh, clean session (grill decision); transcript-carrying fork is post-MVP.
8. No disk-size measurement in v1 (grill decision): counts + evidence-based cleanup suggestions carry footprint legibility; bulk cleanup is post-MVP (grill decision).
9. Automation jobs/triggers keep executing at the workspace root (ADR-003 non-goal). Spawn exposes no user-facing worktree choice in v1, but the spawn request carries the internal inherited `worktree_id` (invariant 20) — inheritance is structural, only the *choice* dimension is deferred.
10. Web selection scoping is client-local (store v3), mirroring workspace selection; the daemon holds no "current worktree".
11. The GitHub extension's `gh` fallback shells `gh auth token` once per capability probe; it never writes `gh` state.
12. Loop `run-loop` nodes forward the parent run's environment; they declare none of their own (US-017 EC-2).
13. The core URL-shape table covers exactly github.com, gitlab.com, and bitbucket.org compare links; unknown hosts without a serving extension fall back to the remote's web URL. An extension's `compare_url_template` always wins.
14. The bundled GitHub extension serves github.com only in v1; enterprise/custom hosts arrive as an extension-side configuration later, with no core change.
15. Adoption is not gated by `worktree.pre_create` in v1 — it mints a record without touching the repository and is reversible via dismiss.
