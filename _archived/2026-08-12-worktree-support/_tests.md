# Test Specification: Native Worktree Support

Canonical test contract for native worktree support. Companion to `_techspec.md`. Derived from `_user_stories.md` (US-001..US-033) and `_techspec.md` (components C1–C15). `cy-create-tasks` assigns each ID to exactly one task; implementers write exactly the assigned cases.

## Strategy

- **Frameworks/harnesses**: Go `testing` with `t.Run("Should …")` subtests, `t.Parallel` by default, `-race`/`CGO_ENABLED=1`; fakes only at I/O boundaries — the scripted fake `GitRunner` (stdout/stderr/exit per invocation) is the unit-level boundary; integration uses **real git** in `t.TempDir()` repositories (`+integration` build tag, co-located); daemon-side E2E uses the Go harness + `acpmock`; web units use Vitest via Turbo from repo root; browser E2E uses Playwright against the daemon-served SPA. HTTP assertions check status code **and** body.
- **Execution**: `make test` (unit) · `make test-integration` · `make test-e2e-runtime` · `make test-e2e-web` · `make bun-test` (web units) · gates via `make gate` / `make gate-full`.
- **Conventions**: table-driven where shapes repeat (porcelain parsing, error mapping, ladder states); no `t.Parallel` on env-mutating tests; integration repos are built per-test with helper builders (`initRepo`, `addWorktree`, `commit`, `bareRemote`); E2E mock/matcher updates co-ship with runtime contract changes; 80% per-package floor. At task-generation time every case is annotated with its invariant, owning layer, and canonical suite — cases extend existing canonical suites (one suite per invariant, never one suite per ID).

## Coverage Matrix

| Source | Behavior | Unit | Integration | E2E |
|---|---|---|---|---|
| US-001 | Create by name; pending→ready; no affordance on non-git | UT-003, UT-010, UT-021 | IT-001 | E2E-006 |
| US-001.EC-1 | Name collision refused, nothing created | UT-023 | IT-006 | E2E-006 |
| US-001.EC-2 | Invalid chars sanitized predictably, previewed | UT-001, UT-002 | — | E2E-006 |
| US-001.EC-3 | Midway failure → full rollback; retry same name works | UT-028 | IT-003 | — |
| US-001.EC-4 | Daemon restart mid-pending → ready or failed, never stuck | UT-084 | IT-005 | — |
| US-001.EC-5 | Concurrent same-name creations → exactly one wins | UT-030 | IT-006 | — |
| US-002 | Explicit branch (no prefix), existing branch, default base | UT-021, UT-022 | IT-002 | — |
| US-002.EC-1 | Branch held by another worktree → refusal names holder | UT-024 | IT-007 | E2E-006 |
| US-002.EC-2 | Branch checked out at root → refusal | UT-025 | IT-007 | — |
| US-002.EC-3 | Base ref missing → refusal names ref | UT-026 | — | — |
| US-002.EC-4 | Unborn repo → plain-language refusal | UT-027 | IT-008 | — |
| US-003 | Session-create environment select | UT-132 | — | E2E-008 |
| US-003.EC-1 | Zero worktrees → root + new only | UT-132 | — | E2E-008 |
| US-003.EC-2 | Worktree removed before submit → fails naming it | UT-093 | — | — |
| US-003.EC-3 | Non-git workspace → control absent | UT-132 | — | E2E-008 |
| US-004 | Composer environment chip; materialize on first send | UT-133 | — | E2E-009 |
| US-004.EC-1 | Switching env preserves draft | UT-133 | — | E2E-009 |
| US-004.EC-2 | Cancel pending materialization → rollback, draft kept | UT-028, UT-133 | — | E2E-009 |
| US-004.EC-3 | Live session → no in-place switch, fork path | UT-133 | — | E2E-010 |
| US-005 | Fork live session → fresh session in worktree | — | — | E2E-010 |
| US-005.EC-1 | Uncommitted changes stay at origin; dialog says so | — | — | E2E-010 |
| US-005.EC-2 | Mid-prompt → fork unavailable with reason | — | — | E2E-010 |
| US-005.EC-3 | Double invoke → one session per confirmation | — | — | E2E-010 |
| US-006 | CLI verbs with structured output parity | UT-121, UT-122 | IT-033 | E2E-003 |
| US-006.EC-1 | Empty list → empty array, exit 0 | UT-122 | — | E2E-003 |
| US-006.EC-2 | Unknown worktree on session start → not-found, no root fallback | UT-090, UT-122 | — | E2E-003 |
| US-006.EC-3 | Racing CLI removals → one wins, other already-removed | UT-083 | IT-024 | — |
| US-007 | Native tools create/list, permission-gated | — | — | E2E-004 |
| US-007.EC-1 | Mode denies mutation → deterministic denial | — | — | E2E-004 |
| US-007.EC-2 | Foreign workspace → cross-workspace denial, no leak | — | IT-038 | E2E-004 |
| US-007.EC-3 | Retried create → idempotent, never two worktrees | UT-030, UT-041 | — | — |
| US-008 | Discovery lists every git-known worktree nested | UT-043 | IT-013 | E2E-005 |
| US-008.EC-1 | Unreadable/unavailable location → explicit state | UT-044 | — | — |
| US-008.EC-2 | Many stale discovered → list stays scannable | UT-134 | — | E2E-005 |
| US-008.EC-3 | Discovery failure → error state, nav still works | UT-047 | — | — |
| US-009 | Adopt-on-select with identity validation, no bootstrap | UT-037 | IT-009 | E2E-007 |
| US-009.EC-1 | Double adoption → one row | UT-041 | IT-011 | — |
| US-009.EC-2 | Directory vanished → deterministic not-found | UT-042 | — | — |
| US-009.EC-3 | Branch claimed by adopted worktree → adoption still succeeds | — | IT-012 | — |
| US-010 | Nested rendering in both surfaces, record labels | UT-128 | — | E2E-005 |
| US-010.EC-1 | No worktree-of-worktree affordance | UT-128 | — | — |
| US-010.EC-2 | Many worktrees → collapse/scroll, parent selectable | UT-134 | — | E2E-005 |
| US-010.EC-3 | Keyboard order predictable | UT-134 | — | E2E-005 |
| US-011 | Selection scopes views; parent shows whole picture | — | — | E2E-005 |
| US-011.EC-1 | Selected worktree removed → fallback to parent + notice | UT-131 | — | — |
| US-011.EC-2 | Selection survives reload when worktree exists | UT-131 | — | — |
| US-011.EC-3 | Two windows keep independent selections | UT-131 | — | — |
| US-012 | Row status only from real state | UT-129 | — | E2E-014 |
| US-012.EC-1 | Detached → "pinned at <sha>", no invented branch | UT-017, UT-129 | — | — |
| US-012.EC-2 | Offline remote values marked stale/hidden | UT-051, UT-129 | — | — |
| US-012.EC-3 | Activity indicator resolves within a refresh | — | IT-034 | — |
| US-013 | Parent-brain inheritance (memory/skills/permissions) | — | IT-028 | E2E-001 |
| US-013.EC-1 | Concurrent sessions in one worktree allowed | — | IT-026 | — |
| US-013.EC-2 | Same logical file, separate checkouts, no interference | — | IT-026 | — |
| US-013.EC-3 | Session context names workspace + worktree binding | — | IT-026 | — |
| US-014 | Task worktree policy on setup surface, all surfaces read back; policy-alone update preserves the rest | UT-094, UT-096, UT-101 | IT-033 | E2E-011 |
| US-014.EC-1 | Removed ref → flagged at read, refused at run start | UT-095, UT-097 | — | — |
| US-014.EC-2 | Ref to other workspace → validation refusal | UT-095 | — | — |
| US-014.EC-3 | Clear setup → back to inherit | UT-103 | — | — |
| US-014.AC-4 | Active run locks policy change, names run | UT-102 | — | E2E-011 |
| US-015 | Per-run fresh worktree, namespaced branch, persists after terminal | UT-098 | IT-029 | E2E-002 |
| US-015.EC-1 | Materialization failure fails run, no orphan | UT-099 | IT-029 | — |
| US-015.EC-2 | Concurrent runs → distinct names, never collide | UT-004 | IT-029 | — |
| US-015.EC-3 | Canceled run → worktree persists, standard removal rules | — | IT-029 | — |
| US-015.EC-4 | Accumulated per-run worktrees stay grouped/legible | UT-134 | — | E2E-002 |
| US-016 | Fan-out per-run isolation | — | IT-031 | E2E-012 |
| US-016.EC-1 | Existing caps unchanged; confirmation states count | — | — | E2E-012 |
| US-016.EC-2 | Partial failure attributable per run | — | IT-031 | — |
| US-017 | Loop-level worktree default + per-run override | UT-105 | — | E2E-013 |
| US-017.EC-1 | Referenced worktree removed → run fails validation | UT-108 | — | — |
| US-017.EC-2 | Child loop inherits parent env unless it declares | UT-110 | — | — |
| US-018 | Single Environment control on run-agent/goal | UT-104, UT-105, UT-109 | IT-032 | E2E-013 |
| US-018.EC-1 | Retired `cwd` fails validation naming the migration (amended — B-009 hard cut) | UT-104 | — | — |
| US-018.EC-2 | Unresolvable template → node fails pre-session | UT-106 | — | — |
| US-018.EC-3 | Fan-out instances follow node env; per-run per branch | UT-107 | IT-032 | — |
| US-019 | Five-field status strip, truthful | UT-049 | IT-015 | E2E-014 |
| US-019.EC-1 | No upstream → ahead/behind absent; publish semantics | UT-019, UT-052 | IT-015 | — |
| US-019.EC-2 | Local immediate; remote later, marked stale | UT-051, UT-063 | — | — |
| US-019.EC-3 | Dirty counts refresh on turn end | — | IT-034 | — |
| US-020 | Computed primary auto-advances; per-action reasons | UT-053, UT-054, UT-055 | — | E2E-014 |
| US-020.EC-1 | No remote → push/PR blocked, commit works | UT-059 | — | — |
| US-020.EC-2 | Detached → branch actions blocked | UT-060 | — | — |
| US-020.EC-3 | Status error blocks whole control | UT-057 | — | E2E-014 |
| US-020.AC-3 | Session executing → all actions paused with reason | UT-056 | — | E2E-014 |
| US-020.AC-4 | Behind/diverged blocks, never auto-resolves | UT-058 | — | — |
| US-021 | Commit step: scope shown (counts + named untracked additions), honest message default | UT-069, UT-152 | — | E2E-014 |
| US-021.EC-1 | Nothing to commit → closes without empty commit | UT-069 | — | — |
| US-021.EC-2 | Commit hooks fail → output shown, no silent retry | UT-070 | — | — |
| US-021.EC-3 | Honest default message (no generation states in v1 — B-013); staged-agent path | UT-069 | — | E2E-014 |
| US-022 | PR step: base chain, template prefill, browser fallback | UT-071, UT-137 | IT-035 | E2E-014 |
| US-022.EC-1 | Multiple templates → abstain, no error | UT-071 | — | — |
| US-022.EC-2 | Unsupported remote → affordances absent, not disabled | UT-063, UT-127 | — | E2E-017 |
| US-022.EC-3 | Zero commits ahead → blocked "no changes" | UT-073 | — | — |
| US-022.EC-4 | Draft as peer choice | UT-072 | — | — |
| US-023 | Streamed progress, per-step outcomes, one CTA | UT-064, UT-065 | IT-034 | E2E-014 |
| US-023.EC-1 | Mid-chain failure attributed; later steps never ran | UT-066 | — | — |
| US-023.EC-2 | Cancel mid-flight → completed steps stand | UT-067 | — | — |
| US-024 | Merged detection two-tier; cleanup suggested never run | UT-061, UT-062 | — | E2E-015 |
| US-024.EC-1 | Newer local commits beat merged verdict | UT-061 | — | — |
| US-024.EC-2 | Stale remote → indicator downgraded | UT-051, UT-061 | — | — |
| US-024.EC-3 | No implicit network requests | UT-046 | IT-013 | — |
| US-025 | Agent-staged exit prompts; execution under permission mode | — | — | E2E-014 |
| US-025.EC-1 | Busy session → staged/queued, never interrupts | — | — | E2E-014 |
| US-025.EC-2 | No bound session → offers to start one | — | — | E2E-014 |
| US-026 | Clean removal: confirm, branch survives, history readable | UT-074 | IT-017 | E2E-015 |
| US-026.EC-1 | Running session → stated, stopped first | UT-082 | IT-026 | — |
| US-026.EC-2 | Partial removal failure → error state + retry | UT-135 | — | — |
| US-026.EC-3 | Removing selected worktree → selection falls back | UT-131 | — | — |
| US-027 | Dirty/unpushed two-step refusal naming the loss | UT-075, UT-076 | IT-018, IT-019 | E2E-016 |
| US-027.EC-1 | Unique-but-on-remote → informational downgrade | UT-077 | IT-019 | — |
| US-027.EC-2 | Safety evaluation failure blocks; read error ≠ clean | UT-078 | — | — |
| US-027.EC-3 | Leftover cleanup verifies directory identity first | UT-081 | IT-021 | — |
| US-028 | Out-of-band removal → missing tombstone, history intact | UT-086 | IT-022 | E2E-016 |
| US-028.EC-1 | Directory restored + identity matches → back to normal | UT-085 | IT-022 | — |
| US-028.EC-2 | Different repo at old path → stays missing | UT-085 | — | — |
| US-028.EC-3 | Parent root vanished → workspace behavior governs; worktree layer deletes nothing | UT-087 | — | — |
| US-029 | Bootstrap: copy list + setup command + visible outcome | UT-031, UT-033 | IT-004 | — |
| US-029.EC-1 | Nothing configured → plain checkout works | UT-035 | — | — |
| US-029.EC-2 | Setup hang → timeout, failed, usable-but-flagged | UT-034 | IT-004 | — |
| US-029.EC-3 | Tracked files never duplicated by copy | UT-031 | IT-004 | — |
| US-030 | Placement/naming config, central default, overrides | UT-111, UT-112 | — | — |
| US-030.EC-1 | Invalid location → failure names the setting | UT-113 | — | — |
| US-030.EC-2 | Change affects future only; nothing moves | UT-112, UT-114 | — | — |
| US-030.EC-3 | Same derived name, two workspaces → no collision | UT-111 | — | — |
| US-031 | Identical structured data across CLI/API/tools | UT-121 | IT-033 | E2E-003, E2E-004 |
| US-031.EC-1 | Pending/missing/error represented in output | UT-121 | — | — |
| US-031.EC-2 | High-frequency polling served from daemon state | UT-046 | — | — |
| US-032 | Forge as extension; GitHub bundled; absent → git-local only | UT-123, UT-124 | IT-035 | E2E-017 |
| US-032.EC-1 | Extension error/rate-limit → degraded fields, git-local fine | UT-125 | — | — |
| US-032.EC-2 | Two extensions claim remote → deterministic, no dupes | UT-127 | — | — |
| US-032.EC-3 | Credentials expire → absent-credential behavior + cause | UT-125 | — | — |
| US-033 | Lifecycle events with identity + origin; bootstrap is same surface | UT-116 | IT-034, IT-037 | — |
| US-033.EC-1 | Fan-out burst → one event per worktree, attributable | — | IT-031, IT-037 | — |
| US-033.EC-2 | Per-worktree ordering; consumer failure never blocks | UT-118 | IT-034 | — |
| C1 naming | Slugging, wordlists, namespace | UT-001–UT-006 | — | — |
| C1 capability/runner | Gate + env contract | UT-007–UT-012 | — | — |
| C1 porcelain | Parsers incl. error paths | UT-013–UT-020 | — | — |
| C1 create/bootstrap | Sequencing, rollback, idempotency | UT-021–UT-036 | IT-001–IT-008 | — |
| C1 adopt/discovery/status | Validation, merge, unknown semantics | UT-037–UT-052 | IT-009–IT-016 | — |
| C1 exit | Plan ladder + actions | UT-053–UT-073 | — | — |
| C1 remove/reconcile | Safety, reclaim, recovery | UT-074–UT-087, UT-135 | IT-017–IT-024 | — |
| C1 lock | Per-repo serialization | — | IT-023 | — |
| C2 store/migration | Fresh/reopen/integrity/equivalence for the worktree migration | — | IT-025 | — |
| C3 session | Containment, binding, resume, sandbox/memory roots | UT-088–UT-093 | IT-026–IT-028 | — |
| C4+C5 task policy | 11 layers + bridge + fingerprint | UT-094–UT-103 | IT-029–IT-031 | — |
| C6 loop | Environment DSL + resolution | UT-104–UT-110 | IT-032 | — |
| C7 events/hooks | Payloads, registry, pre-hook gating, fail-open | UT-116–UT-118, UT-138–UT-140 | IT-037, IT-039 | — |
| C8 API | Error mapping, payload parity, streams | UT-119, UT-120 | IT-033, IT-034, IT-038 | — |
| C9 CLI | Output shapes, exit codes | UT-121, UT-122 | — | E2E-003 |
| C11 config | Defaults/merge/validation/lifecycle | UT-111–UT-115 | — | — |
| C12+C13 forge | Protocol, capabilities contract, credential ladder, conformance | UT-123–UT-127, UT-136, UT-137 | IT-035, IT-036 | E2E-017 |
| C14 web | Partition, chips, ladder, stores, forms | UT-128–UT-134 | — | E2E-005–E2E-017 |
| Peer-review R1 hardening | Workspace predication (B-001), snapshot+lease (B-002), phased create/cancel (B-003), removal fence (B-004), op-id cancel (B-005), strict adoption/paths (B-006), secret non-egress (B-007), commit-scope honesty (B-011), spawn inheritance (B-014), SSE listeners + SplitButton (B-015) | UT-141–UT-152 | IT-040, IT-041 | — |

## Unit Tests

### C1 — Naming (`internal/worktree/naming.go`, TechSpec: Key flows / ADR-011)

- **UT-001** (happy): `DeriveNames("Docs Refresh!")` — returns branch `docs-refresh`, dir `docs-refresh`; same sanitize function feeds both.
- **UT-002** (boundary): `DeriveNames("///___")` — all-invalid input falls back to the documented fallback slug; output deterministic across calls.
- **UT-003** (happy): `GenerateName(taken)` — returns `<adjective>-<noun>` from the embedded wordlists; with the name taken, returns `<name>-2`.
- **UT-004** (happy): `RunBranchName(ns="run/", "fix-flaky-e2e", runID)` — returns `run/fix-flaky-e2e-<suffix>`; suffixes for two distinct run ids differ.
- **UT-005** (error): config validator rejects namespace `"Run Branch/"` with `worktree_config_invalid` naming `worktrees.run_branch_namespace`.
- **UT-006** (state): reclamation eligibility uses the recorded per-branch namespace: a branch created under `run/` stays eligible after config changes to `wt/`; a branch under `wt/` created before the change is not retro-eligible.

### C1 — Capability gate + git runner (`capability.go`, `git_runner.go`, TechSpec: ADR-008)

- **UT-007** (error): `lookPath` returns not-found — every service entry returns `worktree_git_unavailable`; list payload carries the diagnostic instead of erroring.
- **UT-008** (error): version probe returns `git version 2.30.1` — `worktree_git_version_unsupported` with the found version in the message.
- **UT-009** (happy): probe `2.45.0` passes; probe result cached (second call runs no subprocess on the fake runner).
- **UT-010** (happy): workspace root without `.git` — list returns `repo.git_backed=false` and zero worktree data; no git subprocess beyond the probe.
- **UT-011** (happy): real-runner env construction — parent env preserved (incl. `HOME`, `PATH`, `GIT_CONFIG_GLOBAL` if user-set); `GIT_DIR`/`GIT_WORK_TREE`/`GIT_INDEX_FILE` stripped; `GIT_TERMINAL_PROMPT=0` and `GCM_INTERACTIVE=never` present.
- **UT-012** (error): command exceeding its timeout — returns wrapped `context.DeadlineExceeded` with captured stderr in the error string; no goroutine leak (`goleak`-style assertion per repo convention).

### C1 — Porcelain parsers (`git_porcelain.go`, TechSpec: Key flows)

- **UT-013** (happy): `parseWorktreeList` on a 3-entry `-z` fixture (main + two linked, `branch refs/heads/x`) — paths, branches (prefix stripped), main identified positionally.
- **UT-014** (state): entries with `detached` and `prunable gitdir file points to non-existent location` — flags set; prunable reason captured.
- **UT-015** (boundary): `bare` entry skipped from selectable set; final record without trailing separator still flushed.
- **UT-016** (happy): `parseStatusV2` fixture with `# branch.head main`, `# branch.upstream origin/main`, `# branch.ab +2 -1` — branch/upstream/ahead=2/behind=1.
- **UT-017** (state): `# branch.head (detached)` — `detached=true`, branch empty; entries still counted as dirty.
- **UT-018** (happy): numstat merge of staged+unstaged fixtures — file count deduped, insertions/deletions summed.
- **UT-019** (boundary): status without `# branch.ab` (no upstream) — ahead/behind are `nil`, `has_upstream=false`; never zero.
- **UT-020** (error): truncated/garbled porcelain — parser returns a typed parse error; `Status` maps it into `read_error`, not a crash.

### C1 — Create + bootstrap (`create.go`, `bootstrap.go`, TechSpec: Key flows, ADR-006/011)

- **UT-021** (happy): `Create{Name:"docs-refresh"}` — inserts `pending` row, then fake-runner receives, in order: branch existence probe, `branch docs-refresh <default>`, `worktree add <path> docs-refresh`, `config branch.docs-refresh.gh-merge-base <default>`; row ends `ready` with `created_branch=true`, `created_head` recorded, `base_ref` recorded.
- **UT-022** (happy): `Create{ExistingBranch:"feature/x"}` — no `branch` mint, `worktree add <path> feature/x`; `created_branch=false`, no gh-merge-base write.
- **UT-023** (error): name already registered in workspace — `worktree_name_taken`; zero git invocations after the probe.
- **UT-024** (error): requested branch appears in the porcelain list under another worktree — `branch_held_by_worktree` with holder worktree ref in the payload.
- **UT-025** (error): requested branch is the main checkout's — `branch_checked_out_at_root`.
- **UT-026** (error): base ref probe fails — `base_ref_not_found` naming the ref; nothing created.
- **UT-027** (error): `rev-parse HEAD` fails (unborn) — `repo_has_no_commits`; plain-language message.
- **UT-028** (error): `worktree add` exits non-zero after the branch mint — unwind runs `update-ref -d` for the minted branch and deletes the row; a retry with the same name reaches `worktree add` again (no residue).
- **UT-029** (error): target path already exists on disk — `worktree_path_exists`; the existing directory is never touched.
- **UT-030** (idempotency): two concurrent `Create` with one name — store UNIQUE makes exactly one row; loser maps to `worktree_name_taken`.
- **UT-031** (happy): copy-list candidates come from `ls-files --others --ignored --exclude-standard -- <patterns>` only — a tracked file matching a pattern never enters the candidate set.
- **UT-032** (error): destination file exists — copy skips with per-file "skipped (exists)" outcome; no overwrite call issued.
- **UT-033** (happy): setup command env is the exact allowlist — `PATH`, `HOME`, `SHELL`, `LANG`/`LC_*`, `TMPDIR` + `COMPOZY_WORKSPACE_ROOT`, `COMPOZY_WORKTREE_PATH`, `COMPOZY_WORKTREE_ID`, `COMPOZY_WORKTREE_BRANCH`; daemon secrets and `GIT_*` absent; cwd is the new worktree.
- **UT-144** (state): creation records `pending_phase` before each step (`branch → checkout → copy → setup`); phase transitions are durable (visible through the store after each step on the fake runner).
- **UT-145** (error/happy): `CancelCreate` during `pending` rolls back recorded artifacts and deletes the row; on a `ready`/`failed` row it returns `worktree_not_pending`.
- **UT-034** (error): setup command sleeps past `setup_timeout` — killed; `setup_state=failed`, `setup_error` non-empty and readable; worktree still transitions `ready`.
- **UT-035** (happy): no copy list, no setup command — worktree goes `ready` with `setup_state=none`.
- **UT-036** (boundary): a copy candidate resolving inside the target worktree — skipped by the nesting guard.

### C1 — Adopt (`adopt.go`, TechSpec: Key flows, ADR-002)

- **UT-037** (happy): path whose `.git` file's `commondir` resolves to the workspace repo — row minted `{origin:adopted, state:ready}`, name derived from branch; **no** bootstrap invocations on the fake runner.
- **UT-038** (error): path is the main checkout (git dir == common dir) — `adoption_main_checkout`; no filesystem writes.
- **UT-039** (error): common dir resolves to a different repository — `adoption_foreign_repository`.
- **UT-040** (error): `.git` unreadable/absent — `adoption_unreadable`.
- **UT-041** (idempotency): adopting an already-adopted path — returns the existing row, no second row, no event re-fire.
- **UT-042** (error): path stat fails (vanished between list and select) — `worktree_not_found`; no row minted.
- **UT-148** (error): adoption rejects the classified hard cases — candidate absent from strictly parsed porcelain, path-vs-admin `gitdir` linkage broken in either direction, bare, submodule, `core.worktree`-redirected, and symlink-escaping candidates — each with its named refusal; success records the canonical `git_dir` fingerprint on the row.
- **UT-149** (error): creation `path` override outside approved roots — inside a workspace root, inside another worktree, or an ancestor/descendant of either — refuses with `worktree_path_exists`/`worktree_config_invalid` class errors; containment re-validates (canonicalized) immediately before copy, setup, and git mutations.

### C1 — Discovery/list (`discovery.go`, TechSpec: Key flows)

- **UT-043** (happy): merge of 2 registry rows + 4 git entries — discovered set = git entries minus main checkout minus registered paths; canonical-path comparison (symlinked temp dirs equal).
- **UT-044** (state): prunable git entry → discovered item marked stale/non-selectable; unreachable path → explicit unavailable state, not dropped.
- **UT-045** (state): registered row absent from git list and from disk — flips to `missing` exactly once; second reconcile emits no duplicate event.
- **UT-046** (state): within `discovery_cache_ttl` no git invocation on repeat list; `refresh=true` bypasses; mutation invalidates the cache.
- **UT-047** (error): `worktree list` fails — payload carries discovery-error diagnostic; registry rows still returned.
- **UT-048** (boundary): zero rows + zero git entries — empty lists, `repo.git_backed=true`, no error.

### C1 — Status (`status.go`, TechSpec: Key flows)

- **UT-049** (happy): scripted dirty repo — `{dirty_files:3, insertions:10, deletions:2, ahead:1, behind:0, has_upstream:true, branch, head_sha}` persisted with `refreshed_at`.
- **UT-050** (error): status command fails — `read_error` set; previously-known numeric fields cleared to NULL (unknown), not retained as stale-truth.
- **UT-051** (state): payload marks remote-derived fields with their `refreshed_at`/`fetched_at`; consumer contract asserts stale flag derivation past TTL.
- **UT-052** (boundary): no upstream — ahead-vs-base value labeled distinctly from upstream-ahead; behind absent.

### C1 — Exit plan (`exit_plan.go`, TechSpec: Key flows, ADR-004)

- **UT-053** (happy): dirty tree — primary `commit`; menu rows carry per-action reasons ("No uncommitted changes to commit." absent here).
- **UT-054** (happy): clean + ahead with upstream — primary `push`; without upstream — primary `push` with publish semantics flagged.
- **UT-055** (happy): clean + pushed + no PR — primary `open_pr`; with cached open PR — `view_pr` with number/url.
- **UT-056** (state): bound session running — every action `{enabled:false, reason:"Session is executing…"}`; plan carries the global pause cause.
- **UT-057** (state): `read_error` non-empty — whole control blocked by the status failure; no action enabled.
- **UT-058** (state): behind>0 or diverged — push/PR blocked with "Branch has diverged from upstream. Rebase/merge first." / behind reason; no auto-resolve action exists in the plan.
- **UT-059** (state): no remote configured — push/PR blocked with missing-remote reason; commit enabled.
- **UT-060** (state): detached — branch-dependent actions blocked with "create a branch first".
- **UT-061** (happy): merged-precedence table — fresher forge `merged` beats older local; local commits newer than forge verdict beat `merged` (no safe-to-clean); closed-not-merged flips indicator.
- **UT-062** (happy): local evidence — branch with zero unique commits vs base ⇒ safe-to-clean with evidence text; branch present on remote ⇒ downgraded note.
- **UT-063** (state): no forge capability/credential for the remote — PR-state fields and PR actions **absent** from the plan (not disabled); browser compare path present.

### C1 — Exit actions (`exit_actions.go`, TechSpec: Key flows, SD-010)

- **UT-064** (happy): `commit_push` — phases announced up front `[commit, push]`; per-step results recorded; success carries exactly one CTA (`view_pr` when a PR exists, else `open_pr`).
- **UT-065** (happy): push when already up to date — step outcome `skipped` with reason "already up to date".
- **UT-066** (error): push rejected — commit step reported done, push step failed with git stderr, no PR step ran.
- **UT-067** (state): cancel mid-`commit_push` after commit — op ends canceled; commit stands; report names the stop point.
- **UT-068** (error): second `RunExitAction` while one active — `worktree_operation_in_progress`; first op unaffected.
- **UT-069** (happy): empty message — commit uses default `Update N files`; zero staged changes at open — action reports nothing-to-commit and creates no empty commit.
- **UT-152** (state): commit scope payload lists every untracked addition **by name** alongside counts/±; a gitignored sentinel file (e.g. `.env` matching an ignore rule) is neither listed nor staged by the action (B-011).
- **UT-070** (error): commit hook non-zero — hook stdout/stderr surfaced in the step result; exactly one attempt.
- **UT-071** (happy): PR base chain — `branch.<b>.gh-merge-base` wins; absent ⇒ upstream; absent ⇒ repo default; multiple PR templates ⇒ body prefill abstains (no template, no error).
- **UT-072** (happy): `ForgeProvider.CreatePR` returns `opened_existing` — result links the existing PR, no duplicate create call; `draft:true` passes through.
- **UT-073** (error): zero commits ahead of base — `open_pr` blocked with "no changes to include".

### C1 — Remove, reclaim, reconcile, dismiss (`remove.go`, `reclaim.go`, `reconcile.go`, `dismiss.go`, TechSpec: Key flows, ADR-006/007/011)

- **UT-074** (happy): clean worktree — safety evaluation passes, `worktree remove` (no `--force`), row → `removed`, branch untouched, `worktree.removed` dispatched once.
- **UT-075** (error): dirty — `worktree_dirty_requires_force` with `{changed_files, insertions, deletions}` inventory; no git removal invoked.
- **UT-076** (error): unpushed unique commits — `worktree_unpushed_requires_force` with commit count.
- **UT-077** (state): unique commits also on a remote ref — refusal downgrades to informational note; removal proceeds on single confirm.
- **UT-078** (error): safety evaluation status read fails — `worktree_safety_check_failed`; removal blocked; read error never counts as clean.
- **UT-079** (happy): `force=true` — `worktree remove --force` invoked; without the explicit flag `--force` never appears in args.
- **UT-080** (happy): reclamation of unchanged run-branch — `update-ref -d refs/heads/run/x <created_head>` invoked; pointer moved ⇒ reclamation skipped with informational outcome; user branches (`created_branch=false`) never touched.
- **UT-081** (error): leftover directory whose `.git` `gitdir:` pointer resolves outside `<common-dir>/worktrees/` — preserved; original error returned.
- **UT-082** (error): bound session running — removal blocked with `worktree_session_active` until the stop completes.
- **UT-083** (idempotency): removing an already-`removed` worktree — deterministic already-removed result, exit clean.
- **UT-084** (state): boot `RecoverCreations` resolves from the **recorded phase**, never directory existence — phase `branch`/`checkout` ⇒ unwind ⇒ `failed`; phase `copy`/`setup` ⇒ `ready` + `setup_state=failed` ("interrupted by daemon restart"); stale `removing` ⇒ removal decision re-run; stale `running` exit ops ⇒ `failed`; never left `pending`.
- **UT-146** (concurrency): removal opens with CAS `ready → removing` — a session bind, policy resolution, or exit action against a `removing` worktree refuses (≠ ready); refusal/failure transitions back to `ready`; safety evidence is re-read under the repo lock before the destructive call (order: DB fence, then repo lock).
- **UT-147** (idempotency): `CancelExitAction` with a stale/finished `op_id` is a deterministic no-op and can never cancel a later op; the one-active-op rule holds across simulated restart via the durable `worktree_exit_ops` row.
- **UT-085** (state): missing row whose path re-appears — identity match (same common dir) ⇒ back to `ready`; different repo at path ⇒ stays `missing`.
- **UT-086** (happy): `Dismiss` on missing/removed/failed — state `dismissed`; bound sessions/task_runs rows untouched (asserted by store read).
- **UT-087** (state): parent workspace root vanished — worktree reconciler takes no action (workspace-level behavior governs); no worktree row deleted.
- **UT-135** (error): `worktree remove` fails partway (simulated EBUSY) — row keeps a retriable error outcome, state not `removed`; retry path re-runs evaluation.

### C3 — Session binding + containment (`internal/session`, TechSpec: Session binding, ADR-005)

- **UT-088** (happy): unbound session cwd resolves within workspace root (existing behavior pinned); bound session cwd defaults to the worktree root.
- **UT-089** (error): bound session with requested cwd outside the worktree root — `ErrValidation` "escapes root"; the workspace root is NOT an accepted fallback for a bound session.
- **UT-090** (error): binding ref to pending/failed worktree ⇒ `worktree_not_ready`; to missing ⇒ `worktree_missing`; to unknown ⇒ `worktree_not_found` — no session starts at root.
- **UT-091** (state): resume of a bound session re-resolves the binding; worktree now missing ⇒ resume fails `worktree_missing`, session metadata unchanged.
- **UT-092** (happy): bound session start spec — sandbox `LocalRootDir` = worktree root; memory store construction receives the **parent** workspace root; resolved workspace config/agents come from the parent.
- **UT-093** (error): worktree removed between environment selection and submit — session create fails naming the worktree (`worktree_missing`), nothing starts.
- **UT-143** (error): every `Service` read/mutation called with a mismatched `(workspaceID, id)` pair returns `worktree_not_found` with no existence leak; store queries are provably predicated on both keys (B-001).
- **UT-151** (state): spawn of a child from a worktree-bound parent carries the internal inherited `worktree_id` — child resolves through the same binding authority, is contained by the worktree root, and a missing/not-ready inherited binding refuses the spawn; no code path lets a child land at the workspace root (B-014, invariant 20).

### C4+C5 — Task worktree policy (`internal/task`, `internal/daemon`, TechSpec: ADR-009)

- **UT-094** (happy): normalize `{}` ⇒ `{mode:inherit}`; `{mode:"REF ", ref:" x "}` trims/lowercases; validate accepts `ref`+non-empty only.
- **UT-095** (error): `ref` naming a worktree of another workspace or nonexistent — write-time validation refusal with `worktree_ref_invalid`.
- **UT-096** (happy): apply `none` ⇒ opts.CWD = workspace root, no binding; apply `ref` ⇒ opts.CWD = worktree root + `worktree_id` on run and session.
- **UT-097** (error): `ref` resolving to removed/missing at run start — run fails `worktree_ref_invalid`; opts never fall back to root.
- **UT-098** (happy): `per_run` — `MaterializeForRun` called with task slug + run suffix; returned worktree bound to run+session; branch = `<ns><task-slug>-<suffix>`.
- **UT-099** (error): materialization returns error — run fails `per_run_materialization_failed`; unwind invoked; no binding written.
- **UT-100** (state): fingerprint — same profile differing only in worktree mode/ref produces different fingerprints; `per_run` profile never matches an existing role session.
- **UT-101** (happy/state): the worktree-policy patch primitive (`task.Service.SetWorktreePolicy`) sets the worktree block alone and **preserves every other profile block** (worker/coordinator/review/sandbox/participants asserted byte-equal before/after); the same behavior holds through `PATCH /tasks/:id/execution-profile/worktree`, CLI `task profile set-worktree`, and `compozy__task_worktree_policy_set` (US-014 AC-3, B-010); whole-profile PUT keeps replace semantics as a distinct pinned behavior.
- **UT-102** (error): profile write while `CurrentRunID != ""` — refused naming the active run (existing lock extended to the new block).
- **UT-103** (happy): `DeleteExecutionProfile` — subsequent read returns `worktree {mode:inherit}`.

### C6 — Loop environment (`internal/loop`, TechSpec: Key flows)

- **UT-104** (error): definition carrying the retired `params.cwd` — fails validation with the lint reason naming the one-line migration (`params.cwd` → `environment.directory`); nothing executes; the field exists nowhere in source or resolved forms (B-009 hard cut).
- **UT-105** (happy): loop default `worktree ref=X`, node `root` — node wins; node unset — loop default applies; both unset — workspace root.
- **UT-106** (error): `directory` with unresolvable template — node fails validation before any session bind, error names the expression.
- **UT-107** (happy): `per_run` node under a 3-branch fan-out — three distinct materialization calls, three distinct bindings.
- **UT-108** (error): loop-default worktree ref removed before run — run fails validation naming the worktree.
- **UT-109** (happy): `goal` accepts `Environment`; non-agent nodes (`transform`, control, source) reject it at lint.
- **UT-110** (state): `run-loop` node — child loop inherits the parent run's resolved environment unless the child's own config declares one.

### C11 — Config (`internal/config`, TechSpec: Config Lifecycle)

- **UT-111** (happy): zero-config defaults — root resolves to `$COMPOZY_HOME/worktrees`, ns `run/`, timeout `10m`, TTL `30s`; placement layout `<root>/<workspace-name>/<name>` keeps two same-named worktrees of different workspaces apart.
- **UT-112** (happy): workspace overlay `[worktrees] setup_command` overrides global; global keys not overlaid remain; changing config never rewrites existing rows/paths.
- **UT-113** (error): relative `root`, malformed namespace, non-duration `setup_timeout` — each fails validation naming the exact key.
- **UT-114** (state): lifecycle matrix classifies `worktrees.*` and `task.orchestration.profile.default_worktree_mode` as `Live`.
- **UT-115** (happy): profile validation options carry `default_worktree_mode`; `task` package reads it without importing `config`.

### C7 — Events + hooks (`internal/hooks`, `internal/events`)

- **UT-116** (happy): the `worktree` family declares five events — `pre_create`/`pre_remove` with `syncEligible=true`, `created|adopted|removed` with `syncEligible=false`; observe payloads carry `{worktree_id, workspace_id, name, branch, path, origin}`; family validates in both registry spots.
- **UT-138** (error): `worktree.pre_create` hook returns an explicit deny — create refuses with `worktree_denied_by_hook` carrying hook name + reason; no row, no git invocation after the gate.
- **UT-139** (error): `worktree.pre_remove` deny — removal blocked with the hook's reason; payload passed to the hook carries `{worktree, force, risk}`; worktree untouched.
- **UT-140** (state): pre-hook execution error (crash/timeout, no verdict) — operation proceeds (fail-open) and the error is logged; only an explicit deny blocks (invariant 19).
- **UT-150** (error): secret non-egress — a sentinel claim token and a sentinel bound secret never appear in `RunWorktreeRequest` fields, setup/git subprocess env, `setup_error`, `read_error`, refusal payloads, canonical events, or stream frames; diagnostic strings provably pass the shared redaction boundary before persist/append (B-007).
- **UT-117** (happy): canonical registry builds with the worktree entries (names/family/component/outcome present; duplicates panic pinned by the existing registry test).
- **UT-118** (state): hook consumer returns error — lifecycle operation still succeeds; failure logged/reported; dispatch happens at the mutating call site exactly once per transition.

### C8+C9 — API error mapping + CLI output (`internal/api/core`, `internal/cli`)

- **UT-119** (happy): every worktree sentinel error maps to exactly one wire code from the TechSpec table; the mapping test enumerates the table and fails on any unmapped sentinel or duplicate code.
- **UT-120** (error): removal refusal HTTP body — `{code:"worktree_dirty_requires_force", risk:{changed_files,insertions,deletions,unpushed_commits}, downgrade:bool}`; status 409.
- **UT-121** (happy): `worktree list -o json` — fields `{id,name,branch,path,state,origin,dirty,ahead,behind,agent_activity}` match the HTTP payload for the same fixture state, including `pending|missing|error` representations.
- **UT-122** (happy/error): CLI exit codes — success 0 with empty array for zero worktrees; `worktree_not_found` non-zero with structured error payload; no root-fallback message.

### C12+C13 — Forge protocol + GitHub extension (`internal/extension*`, `extensions/forge/github`)

- **UT-123** (happy): remote matching — `https://github.com/o/r.git`, `git@github.com:o/r.git`, uppercase host all normalize and match; non-GitHub remote (including `github.mycompany.com` in v1) returns no capability.
- **UT-136** (happy): `forge/capabilities` payload completeness — for a served remote it returns vocabulary (request noun + action labels), `supports_draft`, `compare_url_template`, template paths, and `credential_source`; exit-plan rendering consumes the vocabulary (no hardcoded "pull request" string in the plan builder).
- **UT-137** (happy/boundary): browser-link ownership — no serving extension + github.com/gitlab.com/bitbucket.org remote ⇒ compare URL from the core URL-shape table; unknown host ⇒ remote web URL fallback; serving extension present ⇒ its `compare_url_template` wins over the table.
- **UT-124** (happy): credential ladder — binding present ⇒ `credential_source:"binding"`; else stubbed `gh auth token` success ⇒ `"gh"`; else capabilities report absent credential and PR methods refuse `forge_unavailable`.
- **UT-125** (error): API 403 rate-limit / 401 expired — `forge_error` with cause `rate_limited`/`credential_expired`; status fields degrade to stale/absent; no retry loop (exactly one HTTP call per gesture).
- **UT-126** (idempotency): `pr_create` when an open PR exists for head — returns `opened_existing` with the PR; create endpoint not called twice (probe-first pinned).
- **UT-127** (state): two enabled extensions both matching a remote — deterministic winner (enable order), winner surfaced in capabilities payload; loser never called for actions.

### C14 — Web units (Vitest, per `_uiux.md` component plan)

- **UT-128** (happy): `groupWorkspaceTree` — worktrees nest under their parent, never as siblings; no nesting affordance under a worktree; non-git workspaces get no worktree group.
- **UT-129** (state): `WorktreeStateChip`/`WorktreeSignals` — chip renders only when state ≠ ready; unknown dirty/ahead render as absent/em-dash (never `+0 −0`); detached renders "pinned at <short-sha>"; stale stamps remote values.
- **UT-130** (happy): `useWorktreeExitLadder` — primary advances `commit → commit&push → push → open_pr → view_pr` from plan payloads; per-row reasons rendered verbatim; read-failure and session-running block the whole control.
- **UT-131** (state): active-workspace store v3 — selecting a worktree scopes context; worktree disappearing from the list falls back to parent with notice; persisted selection restores when present; v2 key ignored.
- **UT-132** (happy): `SessionEnvironmentField` — options = root + ready worktrees + "New worktree…"; resets on workspace change; absent for non-git workspace.
- **UT-133** (state): `SessionEnvironmentChip` — draft-only switching preserves draft text; pending materialization state; failure offers retry/pick-other/continue-at-root keeping the draft; live session shows binding locked (fork affordance).
- **UT-134** (happy): `WorktreeNestList` — locked sort (state group → activity desc), truncation at 5 + "All N" overflow (adopted-only counts), keyboard traversal order matches visual order, discovered rows selectable (adopt gesture), pending/missing inert with reason lane.
- **UT-141** (happy/state): worktree SSE consumers register `addEventListener` for every named event (`worktree_catalog_changed` + per-worktree canonical names) — never `onmessage`-only; invalidation keys are workspace-qualified; listeners and the source are cleaned up on unmount/disconnect (B-015, L-017/L-018 class).
- **UT-142** (happy): `SplitButton` primitive — colocated story + test in `packages/ui` (canonical suite): primary click fires the primary action, chevron opens the menu, `variant`/`disabled` propagate to both segments, keyboard contract (Enter/Space on primary, ArrowDown opens menu) holds.

## Integration Tests

### Worktree lifecycle against real git (`internal/worktree`, `+integration`)

- **IT-001**: init repo with a commit → `Create{Name:"feat-a"}` — real branch + worktree materialize at `<root>/<ws>/feat-a`; `git worktree list` shows it; row `ready`; `created_head` equals branch tip; `branch.feat-a.gh-merge-base` set to the default branch.
- **IT-002**: pre-existing branch `feature/x` not checked out — `Create{ExistingBranch}` checks it out; no new branch; `created_branch=false`.
- **IT-003**: force `worktree add` failure (pre-create the target path) after branch mint — unwind leaves no branch (`show-ref` empty), no row; retry with the same name succeeds.
- **IT-004**: `.env` (gitignored) + tracked `README` + copy_list `[".env*"]` — new tree contains `.env`, does not duplicate `README` via copy; setup command `echo ok > setup.txt` runs in the worktree with the documented env vars; a `sleep 60` setup with `setup_timeout="1s"` yields `setup_state=failed` + readable error while the worktree stays usable.
- **IT-005**: kill service between durable creation phases (simulated) → `RecoverCreations` follows the recorded phase, never directory existence — `branch`/`checkout` ⇒ unwind + `failed` whether artifacts exist or not; `copy`/`setup` with a completed checkout ⇒ `ready` + `setup_state=failed` and an interrupted-restart diagnostic.
- **IT-006**: two goroutines `Create` same name — exactly one `ready` row; other receives `worktree_name_taken`; exactly one directory on disk.
- **IT-007**: branch checked out in another worktree ⇒ refusal names that worktree; branch checked out at root ⇒ `branch_checked_out_at_root` (real git errors classified, not leaked raw).
- **IT-008**: `git init` with zero commits — create refused `repo_has_no_commits`.
- **IT-009**: external `git worktree add ../ext feat-b` → `Adopt(path)` — row `{origin:adopted, ready}`; no bootstrap ran (no copy artifacts); branch/state read truthfully.
- **IT-010**: `Adopt(mainCheckoutPath)` refused `adoption_main_checkout`; `Adopt(otherRepoWorktree)` refused `adoption_foreign_repository`; in both cases target dir content+mtime unchanged.
- **IT-011**: double `Adopt` same path (sequential + concurrent) — one row; second call returns it.
- **IT-012**: discovered worktree on branch `feat-c` while an adopted worktree also exists — adoption succeeds; listing reflects git reality (one checkout per branch).
- **IT-013**: repo with Compozy-created + external + prunable-stale entries — list merge classifies each (`ready`/`discovered`/`stale`); `git worktree list --porcelain` output before/after listing is byte-identical (discovery mutated nothing, no fetch: assert reflog/FETCH_HEAD absent).
- **IT-014**: repeated `List` inside TTL runs no git subprocess (command-count via wrapped runner); `refresh=true` re-runs; `Create` invalidates.
- **IT-015**: status matrix on real states — dirty (counts match `git diff --numstat`), ahead/behind vs a local bare remote, detached HEAD, no-upstream branch — payload values equal git ground truth; unknowns NULL.
- **IT-016**: corrupt `.git/HEAD` in the worktree — `Status` records `read_error`; exit plan blocked; repair restores normal reads.
- **IT-017**: clean removal — directory gone, `git worktree list` no longer shows it, branch still exists, row `removed`, bound (stopped) session row + history still readable via store.
- **IT-018**: dirty worktree removal — first call returns refusal with real counts; `force=true` removes; uncommitted file provably gone (the named loss was real).
- **IT-019**: unpushed unique commits ⇒ refusal with count; after `git push` to the bare remote ⇒ same state downgrades to informational and single-confirm removal proceeds.
- **IT-020**: run-created branch untouched since creation — removal reclaims it via compare-and-delete (`show-ref` empty after); with one extra commit — branch survives removal.
- **IT-021**: stale leftover dir at a removed worktree's path — matching `gitdir:` pointer ⇒ cleaned; foreign `.git` pointer ⇒ preserved and error surfaced.
- **IT-022**: `git worktree remove` executed out-of-band — reconcile flips row to `missing`; sessions/task_runs rows intact; `Dismiss` hides it deleting nothing; restoring the directory (same repo) returns it to `ready`, and an unrelated repo at the same path keeps it `missing`.
- **IT-023**: 8 concurrent mixed ops (create/remove/status) on one repo — mutations serialize (no git "index.lock" failures leak); queue overflow returns `worktree_operation_in_progress`; ops on a second repo proceed concurrently.
- **IT-024**: two racing removals of one worktree — one succeeds, one gets deterministic already-removed; both exit cleanly.
- **IT-025**: migration suite — fresh DB, reopen, ahead-version, integrity, and declarative-equivalence checks cover the worktree migration (tables incl. `worktree_exit_ops`, composite binding FKs, `task_runs` snapshot columns, profile columns, indexes); direct-rejection cases prove the SQL constraints bite: the profile coupled CHECK `((worktree_mode='ref') = (worktree_ref <> ''))`, the exit-ops single-active partial index, and a cross-workspace composite-FK binding insert all fail at the database layer (B-008, B-001).

### Session binding (`internal/session` + daemon wiring, `+integration`)

- **IT-026**: bound session (acp stub) starts with `cmd.Dir` = worktree root; second concurrent session in the same worktree allowed; each of two sessions in different worktrees edits its own checkout (file content diverges, no cross-writes); session context payload names parent workspace + worktree binding; resume after out-of-band removal fails `worktree_missing`; a child spawned from the bound session inherits the binding and containment, and the spawn refuses once the worktree goes missing (B-014).
- **IT-027**: sandbox (local) for a bound session — tool-host file access allows worktree files and refuses paths outside {worktree root}; `LocalRootDir` equals the worktree root.
- **IT-028**: workspace-scoped memory written from a worktree-bound session recalls identically from a root-bound session and vice versa (one workspace brain); catalog rows carry the parent `workspace_id`.

### Task + loop policy (`+integration`)

- **IT-029**: task with `per_run` — enqueue + claim (real queue) materializes a fresh worktree; `task_runs.worktree_id` + `sessions.worktree_id` set; branch `run/<slug>-<suffix>`; terminal run leaves the worktree `ready` with status visible; injected materialization failure fails the run with `per_run_materialization_failed` and leaves no dir/branch/row; concurrent runs of the same task get distinct names; canceled mid-run keeps the worktree.
- **IT-030**: `ref` policy reuses a role session only for the same worktree (fingerprint); switching the profile ref forces a new session; `per_run` never reuses.
- **IT-031**: fan-out of 3 briefs with isolation — 3 runs, 3 distinct worktrees, 3 `worktree.created` events each attributable to its run; failing the 2nd materialization fails only run 2.
- **IT-032**: loop with node `Environment{worktree ref}` and a `per_run` node under fan-out — bind resolves real roots; per-branch instances get distinct worktrees; `Environment{mode:directory}` executes in the templated directory contained by the workspace root.
- **IT-039**: a registered `worktree.pre_create` hook denying creation — manual create refuses with `worktree_denied_by_hook`; a `per_run` task run fails deterministically with the same code and no orphan; observe events (`created`) never fired for the denied attempts.
- **IT-040**: enqueue-time snapshot authority — enqueue a `per_run` run, then flip the profile to `none` before claim: the claimed run still materializes per its snapshot; conversely a run enqueued under `none` ignores a later `per_run` edit. Lease fencing: expire the claim mid-materialization (short lease fixture) — the stale claimant's binding writes are refused by `task.Service`, the worktree unwinds, and the recovered run re-materializes cleanly (B-002).
- **IT-041**: removal race matrix against real git — remove-vs-new-session-bind, remove-vs-exit-action, and remove-vs-same-branch-create interleavings: the `removing` fence makes exactly one side win with a deterministic refusal on the other; an external `git` commit landing between pre-lock evaluation and the under-lock re-check flips the removal to the dirty refusal (TOCTOU closure, B-004).

### API, streams, isolation (`+integration`)

- **IT-033**: HTTP vs UDS full parity matrix — **every** worktree route (list, create, create-cancel, adopt, inspect, status, exit plan, exit action, exit cancel, remove incl. both refusal tiers, dismiss) plus the profile worktree PATCH returns identical JSON on both transports for the same fixture state; CLI `-o json` matches the same shapes; **every** error code from the TechSpec table is exercised at least once and byte-equal across transports (B-015 — a route or code missing from the matrix fails the test by enumeration against the contract table).
- **IT-034**: per-worktree stream + catalog stream — lifecycle transitions (`created`, status refresh, exit steps, `removed`) arrive as durable events; reconnect with `after_sequence` replays without gaps or duplicates; events for one worktree arrive in order; bound-session turn end triggers a status refresh event.
- **IT-035**: forge stub extension (test harness) — conformance for `forge/capabilities`, `forge/status`, `forge/pr_create`; with no extension enabled the exit plan carries no PR fields and the browser tier still works; enabling the stub lights the PR tier.
- **IT-036**: GitHub extension credential ladder — with a bound `GITHUB_TOKEN` secret uses it; without it and a stub `gh` on PATH answering `auth token`, reports `credential_source:"gh"`; with neither, `forge_unavailable`.
- **IT-037**: events coverage matrix — every worktree lifecycle path emits its canonical event; the matrix test fails on a missing emission.
- **IT-038**: two workspaces, each with worktrees — lists, streams, native tool `compozy__worktree_list`, and caches never return the other workspace's worktrees; a session of workspace A calling worktree tools against workspace B is denied by the cross-workspace boundary with no data leak.

## End-to-End Tests

### Runtime (Go harness + acpmock — `make test-e2e-runtime`)

- **E2E-001** (US-001, US-013, US-019–US-021, US-026): create worktree via API → start bound session → agent writes a file → status shows dirty with counts → `exit/actions commit_push` streams phases and lands the commit on the branch (bare-remote asserted) → clean removal with branch surviving and history readable.
- **E2E-002** (US-014, US-015): set worktree policy `per_run` via CLI `task profile update` → create + enqueue task → run executes in a fresh `run/`-branch worktree → terminal state → worktree persists, listed with `origin:per-run` and exit status; policy read-back identical via HTTP, UDS, CLI.
- **E2E-003** (US-006, US-031): script drives `compozy worktree create/list/inspect/status/remove -o json` end-to-end against the daemon — outputs match API state at every step; zero-worktree list returns `[]`; not-found and dirty-refusal exit codes and payloads deterministic.
- **E2E-004** (US-007, US-031): agent session (acpmock) calls `compozy__worktree_create` under `approve-reads` (prompt → approved), `compozy__worktree_list` (scoped to its workspace), and `compozy__worktree_remove` on a dirty worktree — destructive gate + two-step refusal identical to CLI; deny-mode returns deterministic denial without partial creation.

### Web (Playwright — `make test-e2e-web`)

- **E2E-005** (US-008, US-010, US-011): switcher + menubar + overview render the same nested tree (discovered marked, no siblings); selecting a worktree scopes session/task views and the menubar chip reads `workspace / worktree`; selecting the parent shows bindings per item; keyboard-only traversal reaches nested entries.
- **E2E-006** (US-001, US-002): create dialog — name-first with generated placeholder, derived `branch → path` preview, collision refusal, branch-held refusal with "Select that worktree instead", pending row until ready.
- **E2E-007** (US-009): discovered row → select → adoption confirm naming the validation → adopted without bootstrap; refusal variant (metadata resolves into main checkout) names the reason and leaves the row.
- **E2E-008** (US-003): session-create advanced shows the environment select under Workspace with root default + ready worktrees + "New worktree…"; absent on a non-git workspace; resets when workspace changes.
- **E2E-009** (US-004): composer environment chip on a draft — pick "New worktree", send → visible materialization phases → session starts inside; failure path keeps the draft and offers retry / other worktree / root.
- **E2E-010** (US-005): `/worktree` on a live session → fork confirmation states the three facts → new clean session bound to the worktree; original session untouched; mid-turn the action is unavailable with reason.
- **E2E-011** (US-014): task setup sheet — worktree fieldset with the locked mode vocabulary; `ref` picker same-workspace only; invalid ref flagged; locked banner during an active run.
- **E2E-012** (US-016): fan-out dialog isolation checkbox — confirmation states how many worktrees will be created; result attributes per-run outcomes.
- **E2E-013** (US-017, US-018): loop configure worktree default + node inspector single Environment control (no second directory field anywhere); effective environment readout on the node card.
- **E2E-014** (US-019–US-023, US-025, US-012): worktree detail — truthful five-field strip; ladder advances after commit/push; paused-while-session-running state; commit sheet shows scope + honest default-message placeholder; progress toast announces phases, reports a skip reason, ends with one CTA; agent affordance stages a reviewable prompt into the composer (nothing fires), offering to start a session when none is bound.
- **E2E-015** (US-024, US-026): merged indicator (stubbed forge) + safe-to-clean local evidence → "Clean up" runs the standard removal flow; clean confirm names the record label and states the branch is not deleted.
- **E2E-016** (US-027, US-028): dirty removal — refusal names files/± and unpushed commits, offers the exit flow; force confirm re-states the loss; missing-resolution dialog offers Dismiss / It's back with history-preserved copy.
- **E2E-017** (US-022 AC-4, US-032): zero-credential remote — PR cell and PR action rows absent (not disabled); browser compare row present and forms the whole PR step.
