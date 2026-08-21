# Test Specification: Skill Sources

Canonical test contract for configurable skill source folder patterns. Companion to `_spec.md`. Derived from `_user_stories.md` (behavior), `_spec.md` Part II (components), `_dx.md` (CLI/API journeys), and `_uiux.md` (browser journeys).

## Strategy

- Frameworks and harnesses: Go table-driven tests (`t.Run("Should …")`, `t.Parallel` default, `-race`), real temp filesystems for every scan/expose case (no fs fakes); `acpmock` for provider identity and session flows; Vitest for web component states; Playwright (`make test-e2e-web`) for browser journeys. Fakes at I/O boundaries only.
- Execution: `make test` (unit), `make test-integration` (`+integration` tag), `make test-e2e-runtime` (Go harness), `make test-e2e-web` (Playwright). Contract changes co-ship acpmock/matcher updates. `make gate` during iteration; `make gate-full` at close.
- Conventions: status-code **and** body assertions on every API case; deterministic error codes matched exactly; no `t.Parallel` on env-mutating cases; existing suites extended in place (registry, catalog, prompt, settings, CLI, session-thread) rather than new parallel suites.

## Coverage Matrix

| Source | Behavior | Unit | Integration | E2E |
| --- | --- | --- | --- | --- |
| US-001 | Universal folders by default | UT-007, UT-027 | IT-006 | E2E-001 |
| US-001.EC-1 | Absent dir = enabled, zero, no error | UT-056, UT-068 | — | — |
| US-001.EC-2 | Dir without SKILL.md ignored | UT-074 | — | — |
| US-001.EC-3 | Nameless SKILL.md rejected per-skill | UT-075 | — | — |
| US-001.EC-4 | Hundreds of skills → caps + truncated | UT-021 | — | E2E-006 |
| US-002 | Toggle preset, live apply | UT-008, UT-013 | IT-001 | E2E-007 |
| US-002.EC-1 | Disable mid-conversation | — | IT-010 | — |
| US-002.EC-2 | Rapid toggling → last wins | UT-033 | IT-007 | — |
| US-002.EC-3 | Unknown slug rejected w/ suggestion | UT-002, UT-059, UT-067 | — | — |
| US-002.EC-4 | Empty list = compozy only | UT-010 | — | — |
| US-003 | Custom directories | UT-003, UT-004, UT-009, UT-011 | — | E2E-008 |
| US-003.AC-4 | Custom display slug + deterministic collision suffix | UT-004, UT-081, UT-086 | — | — |
| US-003.EC-6 | Unreadable root → explicit state, counts omitted | UT-097, UT-084 | — | — |
| US-003.EC-1 | Custom path absent | UT-056 | — | — |
| US-003.EC-2 | Duplicate root rejected | UT-005, UT-071 | — | E2E-008 |
| US-003.EC-3 | Ws-relative at global scope rejected | UT-006 | — | — |
| US-003.EC-4 | Escaping entry skipped + diagnosed | UT-018 | — | — |
| US-003.EC-5 | Huge tree → truncated | UT-021 | — | E2E-006 |
| US-004 | Committed repo skills for teammates | UT-025 | IT-006 | E2E-001 |
| US-004.EC-1 | Workspace shadows global | UT-025 | — | — |
| US-004.EC-2 | Branch removes dir → skills leave | — | IT-001 | — |
| US-005 | Workspace override | UT-012, UT-057, UT-058 | IT-002, IT-007 | E2E-009 |
| US-005.EC-1 | Empty override = compozy only | UT-010 | IT-002 | — |
| US-005.EC-2 | Concurrent global+ws edits | — | IT-011 | — |
| US-005.EC-3 | Per-key independent override | UT-012, UT-058 | — | — |
| US-006 | Deterministic collision resolution | UT-025, UT-017 | — | — |
| US-006.EC-1 | Resolution recomputes on toggle | — | IT-001 | — |
| US-006.EC-2 | Intra-root duplicate → shadow, not drop | UT-025 | — | — |
| US-006.EC-3 | Skill vs command collision → qualified + diagnostic | UT-043 | — | — |
| US-007 | Symlinked installs appear once | UT-016, UT-017 | IT-006 | — |
| US-007.EC-1 | Dangling link skipped | UT-019 | — | — |
| US-007.EC-2 | Escaping link never followed | UT-018 | — | — |
| US-007.EC-3 | Link cycle detected | UT-020 | — | — |
| US-007.EC-4 | Own expose links never self-shadow | — | IT-009 | — |
| US-007.EC-5 | Cross-workspace link rejected | UT-083 | IT-002 | — |
| US-008 | Picker lists all sources w/ origin | UT-041, UT-072 | — | E2E-002, E2E-010 |
| US-008.EC-1 | Homonyms qualified + distinguishable | UT-041 | — | E2E-002 |
| US-008.EC-2 | Verification failure blocks invocation | — | — | E2E-002 |
| US-008.EC-3 | Source drift rejection | UT-044 | — | E2E-004 |
| US-008.EC-4 | Body over budget | — (existing invocation-budget suite owns; unchanged) | — | — |
| US-009 | Injection suppression | UT-035, UT-036, UT-040 | IT-008 | E2E-003 |
| US-009.EC-1 | Compozy-winner not suppressed | UT-037 | — | — |
| US-009.EC-2 | Custom origin never suppressed | UT-076 | — | — |
| US-009.EC-3 | Unknown provider → no suppression | UT-038 | — | — |
| US-009.EC-4 | Relocated/isolated provider home → no operator-home suppression | UT-085 | IT-008 | — |
| US-009.EC-5 | Hermes global universal folder → injected (per-root asymmetry) | UT-036 | IT-008 | — |
| US-010 | Create + expose | UT-066 | — | E2E-005 |
| US-010.EC-1 | Target disabled | UT-048 | — | — |
| US-010.EC-2 | Name conflict at target | UT-050 | — | — |
| US-010.EC-3 | Link unsupported, no copy | UT-054 | — | — |
| US-010.EC-4 | Absent target root created + rollback removes only self-created empty dirs | UT-087 | — | — |
| US-010.EC-5 | Racing exposes on one target → one winner, no half-state | UT-046 (+ `UNIQUE(link_path)` via IT-015) | — | — |
| US-011 | Expose lifecycle + cleanup | UT-045, UT-052, UT-053 | IT-004 | E2E-011 |
| US-011.EC-1 | Bundled not exposable | UT-047 | — | — |
| US-011.EC-2 | Re-expose idempotent | UT-046 | — | — |
| US-011.EC-3 | Manually deleted link reported + repairable | UT-053 | — | E2E-011 |
| US-011.EC-4 | Moved target → broken + repair | UT-053 | — | E2E-011 |
| US-011.EC-5 | Foreign link refused | UT-051 | — | — |
| US-011.EC-6 | Remove blocked by uncleanable link → abort, canonical preserved | — | IT-013 | — |
| US-011.EC-7 | Foreign conflict reported, no repair affordance | UT-053, UT-073 | — | E2E-011 |
| US-011.EC-8 | Crash mid-expose → missing, re-expose repairs | UT-077 | IT-013 | — |
| US-011.EC-9 | Unexpose repetition idempotent | UT-088 | — | — |
| US-012 | Agent-structured management | UT-063, UT-064 | IT-005 | E2E-006 |
| US-012.EC-1 | Agent scope read-only for source keys | UT-015 | — | — |
| US-012.EC-4 | Agent-scope expose/unexpose allowed (enable/disable gate) | UT-093 | — | — |
| US-012.EC-2 | Concurrent writes sequentialized | — | IT-011 | — |
| US-012.EC-3 | HTTP/UDS parity | — | IT-005 | — |
| US-013 | Settings sources section | UT-068, UT-069, UT-070, UT-071 | — | E2E-007, E2E-008, E2E-009 |
| US-013.EC-1 | Runtime unavailable degradation | UT-068 | — | — |
| US-013.EC-2 | Defaults-only presentation | UT-069 | — | — |
| US-013.EC-3 | Agent scope read-only notice | UT-070 | — | — |
| US-013.EC-4 | Absent-dir row still toggles | UT-068 | — | — |
| US-014 | Visible truncation + collisions | UT-021, UT-043 | — | E2E-006 |
| US-014.EC-1 | Truncation clears on refresh | — | IT-001 | — |
| US-014.EC-2 | Aggregated collision presentation | UT-069 | — | — |
| US-014.AC-4 | Per-root verification counts visible | UT-084 | — | — |
| US-015 | Ecosystem frontmatter loads without warning noise | UT-029, UT-030 | — | — |
| US-015.EC-1 | Recognized fields never enforced | UT-029 | — | — |
| CLI verbs (workspace view, where, info states, failure renderings) | per `_dx.md` transcripts | UT-089–UT-092, UT-098 | — | — |
| Public projections closure (native tools, extension host API, list route) | origin/exposures + digests | UT-094–UT-096 | IT-005 | — |
| Preset table + validation (Part II: Core Interfaces) | slugs, suggestion, auto-slug | UT-001–UT-006 | — | — |
| Root resolution | ordered specs per scope, OwnerScope/WorkspaceID, RootID | UT-007–UT-011, UT-086 | — | — |
| Config overlay/lifecycle/CLI parse | tri-state, Live, forms | UT-012–UT-015 | — | — |
| Scanner | symlink matrix, caps, stats | UT-016–UT-023, UT-074 | — | — |
| Registry loads | tiers, origin, dedup, guards, caches | UT-024–UT-032, UT-075 | IT-006, IT-007 | — |
| Watcher/live apply | roots provider, snapshot branch, generation fence | UT-033, UT-034 | IT-001, IT-012 | — |
| Injection policy | filter, hash order, canonical provider ids, observability | UT-035–UT-040, UT-076, UT-085 | IT-008 | E2E-003 |
| Command catalog | candidate projection, RootID identity, drift, collisions | UT-041–UT-044, UT-081 | — | E2E-002, E2E-004 |
| Expose manager | records, reconcile, path safety, state machines (expose/unexpose/remove), absent-root creation, every error | UT-045–UT-055, UT-077–UT-080, UT-087, UT-088 | IT-004, IT-009, IT-013 | E2E-005, E2E-011 |
| `skill_exposures` schema | fresh/reopen/ahead/integrity/equivalence | — (owned by canonical store suites) | IT-015 | — |
| Settings section + API | envelope, scopes, tri-state wire shapes, error shapes | UT-056–UT-062, UT-082 | IT-005 | — |
| Workspace isolation | cross-workspace containment, cache keys | UT-083 | IT-002 | — |
| Diagnostics read model | labels, counts, skips, collisions, verification | UT-084 | — | E2E-006 |
| Observability contract | canonical events + correlation keys | — | IT-014 (coverage matrix) | — |
| CLI verbs | outputs + failure shapes | UT-063–UT-067 | — | E2E-001, E2E-005, E2E-006 |
| Web components (S1–S3) | states per `_uiux.md` | UT-068–UT-073 | — | E2E-007–E2E-011 |
| Resource-authority republication (Part II risk #1) | configured roots survive sync | — | IT-003 | — |

## Unit Tests

### Preset table + validation (`internal/config`)

- **UT-001** (happy): `SkillSourcePresets()` — returns exactly `agents` (WorkspaceRel `.agents/skills`, GlobalPath `~/.agents/skills`, **WorkspaceNativeReaders** `[openclaw, hermes]`, **GlobalNativeReaders** `[openclaw]`, DefaultOn) and `claude` (`.claude/skills`, `~/.claude/skills`, both reader lists `[claude]`); implicit `compozy` exposed as builtin with `AlwaysOn` and empty reader lists.
- **UT-002** (error): `ValidateSkillSources([]string{"agnets"})` — error message contains `unknown skill source preset "agnets"`, `valid: agents, claude`, suggestion `agents`.
- **UT-003** (happy): `CustomSourceSlugs(["~/team-skills"])` → `team-skills`; sanitizes to `[a-z0-9-]`.
- **UT-004** (boundary): batch allocator with two colliding basenames (`~/a/team-skills`, `~/b/team-skills`) → deterministic short-hash suffixes, identical output for any input order; adding/removing a non-colliding path changes no other slug.
- **UT-005** (error): custom path resolving to an already-active root (e.g. `~/.agents/skills` while `agents` enabled) → `duplicate_skill_source` naming the owning source.
- **UT-006** (error): `./rel/path` in `custom_sources` at global scope → `invalid_source_path` "workspace-relative paths require workspace scope".

### Root resolution (`internal/config`) — owning suite: `internal/config` config/agent tests

- **UT-007** (happy): default config → global roots = [`~/.agents/skills` (agents/user-scope), `$COMPOZY_HOME/skills` (compozy last)]; workspace roots via `SkillsDirs` = [`.agents/skills` (agents), `.compozy/skills` (compozy last)].
- **UT-008** (happy): `sources=["agents","claude"]` → workspace order [agents, claude, compozy-last]; list order preserved for non-compozy.
- **UT-009** (happy): `custom_sources=["~/team-skills"]` at global scope → spec `{Dir: abs, SourceSlug: "team-skills", Kind: custom, OwnerScope: global, WorkspaceID: ""}`; skills-side tier derivation maps it to `SourceAdditional`.
- **UT-010** (state): `sources=[]`, `custom_sources=[]` → compozy roots only, at both scopes.
- **UT-011** (happy): `~` expansion produces absolute dirs (reuses `expandUserPath`; no duplicate expander).
- **UT-086** (state): resolved specs carry `OwnerScope`/`WorkspaceID` correctly per scope (workspace roots → workspace+id; global roots → global, empty id); `RootID()` is stable across process restarts, config list reordering, AND display-slug re-allocation (adding a colliding basename that re-suffixes an existing slug leaves every `RootID` unchanged); differs for same-slug roots in different scopes.

### Config overlay / lifecycle / CLI parse

- **UT-012** (state): workspace overlay — key absent inherits global; `sources` present replaces whole list while absent `custom_sources` still inherits (per-key independence, both directions).
- **UT-013** (happy): lifecycle classification — `skills.sources` and `skills.custom_sources` → `Live`; `skills.enabled` still RestartRequired (rule ordering).
- **UT-014** (happy): CLI value parse — `agents,claude` and `["agents","claude"]` both → same slice; empty string → `[]`.
- **UT-015** (error): settings update at agent scope touching `sources` → rejected with the section's scope-policy error; `disabled_skills` at agent scope still accepted.

### Scanner (`internal/skillscan`)

- **UT-016** (happy): root containing symlinked dir `foo` → real dir with SKILL.md elsewhere-in-trusted-union → skill candidate loaded via link.
- **UT-017** (happy): same realpath reachable from two scanned roots → single catalog entry attributed to the higher-precedence root; no shadow record between the two aliases.
- **UT-018** (error): first-level link resolving outside the trusted union (`/etc/target`) → skipped, `SkippedLink{Reason: escape}`; root scan continues.
- **UT-019** (error): dangling link → `SkippedLink{Reason: dangling}`.
- **UT-020** (error): link cycle (a→b→a) → `SkippedLink{Reason: cycle}`; scan terminates normally.
- **UT-021** (boundary): root with candidates over the 300 cap → `RootScanStats.Truncated=true`, count == scanned subset.
- **UT-022** (state): second-level (nested) symlink NOT followed — unchanged posture.
- **UT-023** (boundary): root under a symlinked parent (macOS `/private/var` style) → containment check canonicalizes source root before compare; skill loads.
- **UT-074** (state): subdirectory without `SKILL.md` → no candidate, no diagnostic.
- **UT-083** (error): per-projection containment — a first-level symlink in workspace A's root resolving into workspace B's configured root is rejected with an `escape` skip diagnostic; the same link target inside A's own roots loads.
- **UT-097** (error): unreadable root — an existing directory with permissions denied → `RootScanStats` reports `readable: false`, zero candidates, no fatal error; sibling roots in the same pass load normally; restoring permissions yields candidates on the next scan.

### Registry loads (`internal/skills`)

- **UT-024** (happy): global load over `[agents-global, compozy-global]` → agents skills tier `user` with `origin="agents"`; compozy skills `origin=""`.
- **UT-025** (happy): workspace load — same name in `.agents/skills` and `.compozy/skills` → compozy wins (loaded last), loser recorded via `overlaySkill` shadow with both paths inspectable.
- **UT-026** (state): custom root skill → tier `additional`, `origin="team-skills"`.
- **UT-027** (happy): `origin` present on every absorbed skill in list/read models; empty for compozy tier roots.
- **UT-028** (error): absorbed skill failing `VerifyContent` critical → blocked with `verification_failed` diagnostic; sibling skills unaffected.
- **UT-029** (happy): frontmatter with `allowed-tools`, `context: fork`, `hooks`, `model` → zero warnings; fields not acted on.
- **UT-030** (error): frontmatter with `total_nonsense_field` → warning still emitted.
- **UT-031** (state): `workspaceSkillPathsMatchRoots` accepts resolved-spec paths (fast path kept) and rejects a stale path from a removed source (falls to slow path, correct result).
- **UT-032** (state): workspace cache key differs across source-config generations; old generation entry not served after change.
- **UT-075** (error): SKILL.md without `name` → per-skill rejection diagnostic; root continues.

### Watcher / live apply

- **UT-033** (state): roots provider re-derivation — config change to `sources` changes the polled root set on next `pollOnce`; removed root's skills drop after refresh.
- **UT-034** (state): `watcherSnapshotRoot` on a custom root ending in `/agents/skills-like` path (basename `agents`) takes the skills branch, not the agents-dir branch (regression for the basename check).

### Injection policy (`internal/daemon` harness)

- **UT-035** (happy): `claude` session (operator home, no overrides), skill winning root = `~/.claude/skills` → excluded from startup section A and per-turn catalog B; present in command-catalog projection.
- **UT-036** (happy): `openclaw` session, winning root = workspace `.agents/skills` → excluded from A/B; `hermes` session with winning root = **global** `~/.agents/skills` → NOT excluded (Hermes doesn't read that root natively); `hermes` with workspace `.agents/skills` winner → excluded.
- **UT-037** (state): same name absorbed from claude root but shadowed by compozy winner → injected (keying on winning root, not name).
- **UT-038** (state): empty/unknown `Provider` on session → no filtering.
- **UT-039** (ordering): augmenter sha256 computed over the filtered catalog — enabling suppression changes the signature, unchanged-short-circuit still fires on repeat turns.
- **UT-040** (happy): each suppression emits the observability tag/diagnostic `{skill, source, provider}`.
- **UT-076** (state): custom-origin skill never suppressed for any provider (`native_readers` empty).
- **UT-085** (state): session-native-root resolution — canonical alias mapping (`claude-code → claude`, `hermes-agent → hermes`); `hermes` session resolves workspace `.agents/skills` as native but NOT `~/.agents/skills`; `CLAUDE_CONFIG_DIR`/`HERMES_HOME` overrides relocate the resolved global native root (operator-home root no longer matches → not suppressed); `home_policy = isolated` yields no operator-home native roots; unknown/custom provider ids → empty native-root set; classification is typed (no command-text or alias-format matching).

### Command catalog

- **UT-041** (happy): qualified descriptors — `agents:frontend-qa`, `claude:pdf-tools`, `team-skills:audit`; bare token for unambiguous names.
- **UT-042** (state): `sameSkillSource` — Source identity stable between catalog build and `Expand` within one generation; expansion succeeds.
- **UT-043** (error): bare-name collision (skill vs builtin, skill vs skill across roots) → loser listed in diagnostics with its qualified form; never silently absent.
- **UT-044** (error): source disabled between build and expand (generation bump) → `ErrUnavailable`-class drift rejection with existing shape.
- **UT-081** (state): pre-overlay candidate projection — two physical homonyms across roots → bare command resolves the winner AND both qualified commands expand their exact bodies (resolved by `RootID`, not display slug); custom slug hash-suffix stable under config reordering.

### Expose manager (`internal/skills/expose.go`) — owning suite: new canonical `internal/skills/expose_test.go` (new component)

- **UT-045** (happy): `Expose(skill, ["agents"])` — record inserted (with `LinkTarget`), then symlink at `<ws>/.agents/skills/<name>` → canonical dir; returns `TargetResult{ok, Exposure.Status: healthy}`.
- **UT-046** (idempotency): repeat expose over healthy link → success, no fs mutation (mtime unchanged), record `updated_at` untouched or monotonically bumped — asserted one way.
- **UT-047** (error): bundled skill → `skill_not_exposable`, message points to copy-via-create.
- **UT-048** (error): target preset disabled → `expose_target_disabled` naming enabled targets.
- **UT-049** (error): target names a custom source → `expose_target_invalid`.
- **UT-050** (error): target path occupied by foreign dir/file → `expose_name_conflict`; target untouched.
- **UT-051** (error): `Unexpose` on a same-name link resolving elsewhere → `expose_foreign_link`; link preserved.
- **UT-052** (happy): `Unexpose` removes only the CompozyOS link; canonical dir intact.
- **UT-053** (state): `Exposures()` status matrix — link manually deleted → `missing`; link intact but canonical moved/deleted (Readlink == recorded `LinkTarget`) → `broken`; path occupied by a different symlink or a regular dir → `foreign_conflict`; intact → `healthy`.
- **UT-054** (error): symlink syscall failure (permission-denied fixture) → `expose_link_unsupported`; asserts **no copy** was created at target.
- **UT-055** (state): skill removal cleans every CompozyOS-created link across targets (record set is the cleanup target list, including links under a now-disabled preset).
- **UT-077** (state): exposure records + reconcile — record with live resolving link → `healthy`; record whose link was deleted → `missing`; record whose link still has `Readlink == LinkTarget` but no longer resolves (canonical moved/deleted) → `broken`; record whose path holds a link with `Readlink != LinkTarget` (retargeted by someone else) → `foreign_conflict`, never adopted or mutated; same-name link with **no record** → reported foreign, never adopted. (UT-053 remains the canonical four-state matrix; this case owns the record-side reconcile.)
- **UT-078** (error): `validateExposeName` matrix — absolute path, `/` and `\` separators, `..`, encoded traversal (`%2e%2e`), NUL byte, Unicode normalization tricks, over-length → each rejected with `unsafe_skill_name`.
- **UT-079** (error/boundary): `resolveExposeDest` — deepest-existing-parent containment; symlinked parent directory resolving outside the preset root → rejected; macOS `/private/var` canonicalization case passes.
- **UT-080** (error): multi-target semantics — preflight failure on any target → zero mutations anywhere; mid-sequence commit failure → completed targets rolled back (records + links gone) with `rolled_back: true`; rollback failure → per-target cleanup errors surfaced and `skills.exposure.cleanup_failed` emitted.
- **UT-087** (state): absent enabled preset root — expose creates exactly the preset root path (containment-proven), succeeds; multi-target failure after root creation → rollback removes the created link/record AND the created directory only if still empty; a directory that gained foreign content meanwhile is left in place.
- **UT-088** (state): unexpose order + crash convergence — link removed before record; a simulated crash between the two steps leaves record-without-link (`missing`), and re-running unexpose deletes the record (idempotent); a record is never deleted while its proven link still exists.

### Settings section + API handlers

- **UT-056** (happy): envelope `sources[]` — builtin+preset+custom rows with `roots[]{path, exists, skill_count, truncated}`; absent dir → `exists: false`, count 0.
- **UT-057** (state): workspace scope PATCH accepted for `sources`/`custom_sources` only; `marketplace.registry` at workspace scope rejected.
- **UT-058** (state): `inherits{sources: false, custom_sources: true}` when only presets overridden.
- **UT-059** (error): PATCH unknown slug → 400 body exactly `{"error":{"code":"unknown_skill_source",...,"valid":["agents","claude"],"suggestion":"agents"}}`.
- **UT-082** (state): workspace PATCH tri-state wire shapes decoded through the shared presence-aware wrapper — absent field untouched (`Present=false`); `null` clears override (`Present=true, Null=true`); empty array sets empty override; non-empty array sets; wrong JSON type → 400 validation error; non-source field present → 400 `workspace_scope_field_forbidden` naming the field; global PATCH keeps full-config shape; HTTP and UDS decode identically (shared parser).
- **UT-060** (happy): `POST /api/skills/{name}/expose` (workspace-scoped) → 200 `{name, workspace_id: "<canonical>", results:[{target, ok: true, exposure:{target, path, status: "healthy"}}], rolled_back: false}`; global-skill operation → `workspace_id` absent from the body.
- **UT-061** (error): single-target conflict → 409 with the ONE failure envelope: top-level `expose_failed`, `workspace_id` placed identically to the success shape, `results[0].error.code == "expose_name_conflict"` with occupying path; multi-target variant carries `rolled_back: true` and a `rolled_back` per-target code on the compensated target.
- **UT-062** (happy): `GET /api/skills/{name}` carries `origin` and `exposures[]{target, path, status}`; empty for compozy-native unexposed skill.
- **UT-084** (happy): sources read model carries `label`, `readable`, `scanned_count`, `skill_count`, `skipped_links[]`, `collisions[]{name, winner_root_id, qualified_form}`, `verification{blocked, warned}` per root; runtime unavailable OR `readable: false` → counts omitted (not zero).
- **UT-091** (happy/error): `POST /api/skills/{name}/unexpose` — 200 with per-target **independent** results (one removed, one `expose_foreign_link`), NO `rolled_back` field in the unexpose envelope; repeat call converges idempotently.
- **UT-093** (happy): agent-scope authorization split — expose/unexpose invoked by an agent-scope caller succeeds (same gate as skill enable/disable); the same caller writing `skills.sources` is rejected (contrast with UT-015).

### CLI verbs

- **UT-063** (happy): `compozy skill sources` — golden table incl. `always on` compozy row, `truncated` note, scope footer.
- **UT-064** (happy): `compozy skill sources -o json` — schema per `_dx.md` (slug/kind/enabled/always_on/roots/native_readers).
- **UT-065** (happy): expose/unexpose human outputs incl. idempotent "already exposed … (no change)".
- **UT-066** (happy): `skill create x --expose agents` → created + exposed lines; fs link exists.
- **UT-067** (error): `config set skills.sources agnets` → exit 1; `-o json` error body matches `_dx.md`.
- **UT-089** (happy): `skill sources --workspace <name>` — human header `scope: workspace (<name>) · overrides: … · inherits: …`; `-o json` carries `workspace_id` + `inherits{}` at workspace scope and omits both at global scope.
- **UT-090** (happy): `skill where` — golden output for a homonym fixture: WINNER line, ALSO lines with tier·source and qualified-form hint, LINKS line for an exposure; empty case renders `— none —`.
- **UT-092** (state): `skill info` EXPOSED TO rendering — one golden per status: `(healthy)`, `(missing — re-expose repairs)`, `(broken — …)`, `(foreign conflict — not our link; no action)`.
- **UT-098** (error): expose CLI failure renderings — per-target lines under `Error: expose failed …` for target-disabled, name-conflict, and multi-target-rolled-back fixtures (exit 1 each); `create --expose` partial success prints the created line AND the expose error, exits 1, and the skill exists on disk.

### Public projections closure (`internal/tools/builtin`, `internal/extension`, `internal/api/core`) — owning suites: existing native-tool, extension host API, and handlers suites

- **UT-094** (happy): native tools — `compozy__skill_list` and `compozy__skill_search` items carry `origin`; `compozy__skill_view` header carries `origin` + `exposures[]` per the `_dx.md` pair; descriptor/schema digests refreshed and asserted against goldens.
- **UT-095** (happy): extension Host API `handleSkillsList` result items carry `origin` via the shared skill summary DTO; `extension/contract/skills.go` type compiles and round-trips the field.
- **UT-096** (happy): `GET /skills` list route items carry `origin` (empty for compozy-tier); body-level assertion on both transports via the IT-005 parity fixtures.

### Web components (Vitest, states per `_uiux.md`)

- **UT-068** (happy/state): `SettingsSkillSourcesSection` renders daemon rows — always-on compozy without switch, counts, absent-dir state, runtime-unavailable suppressing counts while toggles stay enabled.
- **UT-069** (state): truncated warning marker on root line; defaults-only `Empty` when no optional source enabled; aggregated collision diagnostics presentation.
- **UT-070** (state): workspace scope — inherited vs overridden per key + switch-back control; agent scope renders read-only with policy notice.
- **UT-071** (error): custom editor — duplicate entry inline error; scope-invalid path inline error; add/remove flows; section-level save rejection (validation or transport) renders the daemon error verbatim, preserves the draft, applies nothing.
- **UT-072** (state): picker rows — origin label only on non-compozy skills (assert absent on native).
- **UT-073** (state): `SkillExposePanel` — no-exposures default; healthy list; missing/broken danger states WITH repair actions; foreign-conflict info state with ZERO action affordances; multi-select target picker limited to enabled presets; in-flight pending state; partial-failure rendering of per-target `results[]` with rolled-back marking; affordance absent for bundled skill.

## Integration Tests

### Live apply and discovery

- **IT-001**: PATCH `sources` add `claude` → watcher poll → registry refresh: counts in settings envelope update, new skills listed; then remove skill dir on disk → next poll drops it; truncation flag clears after trimming an over-cap root. No daemon restart anywhere.
- **IT-006**: real fs with vercel-labs layout (canonical `~/.agents/skills/foo`, link `~/.claude/skills/foo`, both presets on) → `foo` once, `origin=agents` (higher-precedence reach), plus repo workspace `.agents/skills` skill visible in that workspace only.
- **IT-007**: workspace override change → generation-keyed cache invalidation: projection before/after differs without restart; other workspace projection untouched.
- **IT-010**: disable a source mid-session — previously expanded `<invoked-skills>` content remains in transcript; next turn's injected catalog omits the source's skills; picker refresh drops them.

### Workspace isolation and merge

- **IT-002**: two workspaces, one with override `["agents","claude"]`, one inheriting `["agents"]` → disjoint effective roots; each envelope reports correct `inherits`; skills from claude visible only in the overriding workspace.
- **IT-011** (concurrency): concurrent global PATCH and workspace PATCH → both commit under the sequential-write discipline; effective workspace view equals override; no partial/interleaved list observed in either envelope.

### Resource authority (spec risk #1)

- **IT-003**: enable source → skills present → trigger resource publication (`DiscoverGlobal`/`DiscoverWorkspace` → `ApplyResourceRecords`) → resource-backed projection still contains the absorbed skills with correct origin; disable source → republication removes them.

### Expose lifecycle

- **IT-004**: marketplace install → expose to `agents` → marketplace update preserves link (healthy) → marketplace remove deletes link; no orphan left.
- **IT-009**: exposed skill (compozy canonical + agents link) with both sources scanned → exactly one catalog entry, `origin=""` (compozy), zero self-shadow records.

### Transport parity

- **IT-005**: `GET/PATCH /settings/skills` (both scope shapes) and expose endpoints over HTTP and UDS → byte-equivalent field sets and error codes for the same fixtures.

### Session suppression (integration)

- **IT-008**: session-resolved native-root suppression matrix through real prompt assembly — `claude` session (operator home): claude-origin winner excluded from sections A+B, present in the commands endpoint, explicit `/<skill>` invocation still injects; `hermes` session: workspace `.agents` winner excluded, **global** `~/.agents` winner included; `CLAUDE_CONFIG_DIR` override pointing elsewhere: operator-home claude root NOT suppressed; `home_policy = isolated`: nothing suppressed from operator-home roots; canonical alias resolution (`claude-code`, `hermes-agent`) behaves identically to canonical ids; unknown provider: nothing suppressed. Supersedes the placeholder reference — this is the owning integration case for ADR-009.

### Generation fence and schema

- **IT-012** (concurrency): deterministic live-apply race — block generation N's scan/republication until generation N+1 fully commits, then release N → every read surface (registry snapshot, resource-backed projection, settings envelope, session-command revision) still reflects N+1; N's work is discarded with a supersession event.
- **IT-013**: exposure reconcile lifecycle — manual link deletion → `missing` reported on inspection surfaces; re-expose repairs (record updated, link recreated); skill removal with one link already gone → remaining links cleaned, records cleared, no error; removal with an **uncleanable** link (permission-denied fixture) → aborts with `skill_remove_blocked`, canonical dir and records intact, and re-running after the fix completes removal; reconcile re-run → idempotent no-op.
- **IT-014**: observability coverage matrix — walks every lifecycle path (sources applied, **superseded** (distinct event, never `applied`), apply_failed, truncation, skipped link, exposure created/removed/operation_failed/broken/cleanup_failed) and fails if any canonical event is missing or lacks the full base correlation keys (`workspace_id?`, `config_generation`, `actor_kind`, `actor_id`); asserts durable append precedes the live revision/SSE broadcast for the same change.
- **IT-015**: `skill_exposures` migration — extend the canonical fresh/reopen/ahead/integrity/equivalence store suites for the new table (fresh apply, reopen with data preserved, ahead detection, `atlas.sum` integrity).

## End-to-End Tests

### CLI journeys (Go harness + acpmock; `_dx.md` transcripts verbatim)

- **E2E-001**: golden path — populated `~/.agents/skills` fixture home → `compozy skill list` shows origins → `config set skills.sources agents,claude` prints "applied live" → `skill list --source user` includes claude skills.
- **E2E-005**: expose journey — `skill create review-checklist --expose agents` → fs link assert → `skill info` shows EXPOSED TO healthy → `skill unexpose` → link gone, info updated.
- **E2E-006**: managing-agent journey, start to finish — `skill sources` human + `-o json` against a fixture with absent dir, custom root, and over-cap root (states, counts, `truncated: true` all present) → agent replaces the truncated custom root via `config set skills.custom_sources` → re-reads `skill sources -o json` and confirms the flag cleared and counts updated, no restart (spec UX journey 4).

### Session journeys (Go harness + acpmock)

- **E2E-002**: session commands endpoint lists absorbed skill with origin → submit `/pdf-tools split this` → prompt carries `<invoked-skills>` with verified content; homonym fixture invokes via qualified `claude:pdf-tools`; a verification-failing fixture skill is rejected with the deterministic error.
- **E2E-003**: sessions under mock providers `claude-code` / `openclaw` / unknown → injected catalog excludes native-read origins per provider, commands endpoint always includes them; suppression diagnostics recorded.
- **E2E-004**: pick skill → disable its source via PATCH → submit → drift rejection with existing error shape; no partial injection.

### Browser journeys (Playwright, per `_uiux.md`)

- **E2E-007**: Settings > Skills → toggle claude on → inline "applied immediately" → count appears on the row; toggle off → skills leave the picker (assert via composer).
- **E2E-008**: add custom source `~/team-skills` → row with count appears; adding duplicate shows inline `duplicate_skill_source` error; remove → skills drop.
- **E2E-009**: switch scope to workspace → inherited indicators; override presets → "overridden" chip + other-workspace check via API stays inherited; switch back to inherit.
- **E2E-010**: open session composer, type `/` → absorbed skill row visible with origin label; native skill row without label.
- **E2E-011**: skill detail → expose to agents → exposures block healthy; delete the link on disk (fixture hook) → reload shows **missing** with repair actions; re-expose repairs to healthy; replace the link with a foreign symlink (fixture hook) → reload shows **foreign_conflict** with no action affordances.
