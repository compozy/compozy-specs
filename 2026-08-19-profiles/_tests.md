# Test Specification: Profiles

Canonical test contract for Profiles. Companion to `_spec.md`. Derived from `_user_stories.md` (US-001..033), `_spec.md` Part II (components, safety invariants), `_dx.md` (CLI/API journeys — invocations verbatim), and `_uiux.md` (browser journeys S1–S13).

## Strategy

- Go: table-driven, `t.Run("Should …")` + `t.Parallel` default, `-race`/`CGO_ENABLED=1`; fakes only at I/O boundaries; integration under the `+integration` tag, co-located; migration suites extend the canonical fresh/reopen/ahead/integrity/equivalence harness for the globaldb stream (`00078`, `00079`) and for the memory stream's migration (values + durable move op).
- E2E runtime: daemon-side Go harness (`make test-e2e-runtime`) against `acpmock`, driving the real CLI/UDS/HTTP surfaces with the `_dx.md` transcripts.
- E2E web: Playwright (`make test-e2e-web`) against the daemon-served SPA; no `force: true`.
- Placement: profile-resolution invariants join `internal/cli/workspace_test.go` / `session_test.go` (canonical suites — no new standalone resolution file); status-code **and** body assertions on every API case. Every case below lands in the canonical suite of its owning layer — the per-section "Canonical suite" lines are the placement contract; `cy-create-tasks` maps IDs to those suites, and a new standalone suite is justified only where no existing suite can own the invariant. Generated artifacts are never frozen by bespoke tests (`make codegen-check` owns drift).

## Coverage Matrix

| Source | Behavior | Unit | Integration | E2E |
| --- | --- | --- | --- | --- |
| US-001 | create profile w/ identity | UT-001..004 | IT-031 | E2E-001, E2E-014 |
| US-001.EC-1 | invalid name rejected | UT-002 | — | E2E-002 |
| US-001.EC-2 | name taken (incl. archived) | UT-003 | IT-031 | — |
| US-001.EC-3 | reserved names rejected | UT-004 | — | E2E-002 |
| US-001.EC-4 | create under "All" exits aggregate | — | — | E2E-015 |
| US-001.EC-5 | create race → one winner | — | IT-032 | — |
| US-001.EC-6 | half-created repair by edit | UT-005 | IT-031 | — |
| US-002 | edit identity anytime | UT-006 | — | E2E-014 |
| US-002.EC-1 | invalid color inline reject | UT-007 | — | — |
| US-002.EC-2 | edit archived allowed | UT-008 | — | — |
| US-002.EC-3 | concurrent edit last-write-wins | — | IT-033 | — |
| US-003 | rename tiered handling | UT-009..011 | IT-034, IT-035 | E2E-016 |
| US-003.EC-1 | rename to taken name | UT-003 | — | — |
| US-003.EC-2 | rename `default` refused | UT-010 | — | E2E-003 |
| US-003.EC-3 | unreachable ws skipped→dormant | — | IT-035 | — |
| US-003.EC-4 | declined repo → dormant+hint | — | IT-035 | E2E-016 |
| US-003.EC-5 | rename during running session | — | IT-034 | — |
| US-003.EC-6 | rename back wakes dormant | UT-046 | IT-035 | — |
| US-004 | archive with work | UT-012 | IT-036 | E2E-017 |
| US-004.EC-1 | running sessions block | UT-013 | IT-036 | E2E-003 |
| US-004.EC-2 | archive `default` refused | UT-010 | — | — |
| US-004.EC-3 | automations paused, listed | — | IT-036 | E2E-017 |
| US-004.EC-4 | archive idempotent | UT-014 | — | — |
| US-004.EC-5 | declared profile never resurrected | UT-060 | IT-052 | — |
| US-004.EC-6 | queued runs freeze/resume on archive | UT-087 | IT-067, IT-068 | — |
| US-008 (guard) | archived owner cannot receive new roots | — | IT-076 | — |
| US-027 (read paths) | memory FTS/cache isolation + aggregate refusal | — | IT-074 | — |
| Phase-0 negatives | assert-empty families abort loudly | — | IT-075 | — |
| Availability gate (Invariant 17) | pending op ⇒ unavailable everywhere | UT-093 | IT-072, IT-077 | — |
| Name/path reservations (Invariant 19) | pending ops reserve names/paths | — | IT-080 | — |
| Enablement hard cut (delete target 11) | old global columns die; default rows migrate | — | IT-081 | — |
| MCP profile layers (Config Lifecycle) | sidecar merge/write/orphan | UT-092 | IT-043 | — |
| Notifications vs archive (Invariant 18) | owner-active permit: acquire → hold through ack/advance → crash replay | — | IT-078 | — |
| Extension secret bindings rebuild (delete target 13) | profile dimension + precedence + cleanup | UT-066 | IT-082 | — |
| Session creation witness (delete target 14) | profile id in creation digest | — | IT-083 | — |
| Credential requirements (needs-setup authority) | durable rows; vault write clears; survives update/uninstall | — | IT-049, IT-085 | E2E-008, E2E-023 |
| Total removal catalog (US-006/Invariant 7) | preview == applied, all 8 kinds | UT-015 | IT-038 | E2E-003 |
| Event coverage gate | lifecycle events in canonical matrix | — | IT-079 | — |
| Grant lattice (D24) | closures + meets pinned, no widening | UT-050, UT-051 | — | — |
| US-005 | unarchive restores | — | IT-037 | E2E-017 |
| US-005.EC-1 | name reserved while archived | UT-003 | — | — |
| US-005.EC-2 | unarchive idempotent | UT-014 | — | — |
| US-006 | delete empty + enumerate | UT-015 | IT-038 | E2E-003 |
| US-006.EC-1 | delete `default` refused | UT-010 | — | — |
| US-006.EC-2 | delete vs create race | — | IT-032 | — |
| US-006.EC-3 | deleted declared profile not recreated | UT-060 | IT-052 | — |
| US-006.EC-4 | overrides removed with profile | — | IT-038 | — |
| US-006.EC-5 | remembered entries swept on delete | UT-020 | IT-029 | — |
| US-007 | default invisible until plural | UT-016 | — | E2E-013 |
| US-007.EC-1 | all archived → default-only degrade | — | IT-037 | — |
| US-007.EC-2 | switcher birth on second profile | — | — | E2E-013 |
| US-008 | work lands in active profile | UT-017 | IT-011..013 | E2E-001 |
| US-008.EC-1 | archived automation creates nothing | — | IT-036 | — |
| US-008.EC-2 | two terminals two profiles | — | — | E2E-004 |
| US-008.EC-3 | agent cannot re-aim ownership | UT-074 | IT-066 | — |
| US-008.EC-4 | bridge delivery → instance owner | — | IT-014 | — |
| US-009 | scoped reads only | — | IT-011..016 | E2E-001 |
| US-009.EC-1 | worktrees visible everywhere, tagged | — | IT-015 | — |
| US-009.EC-2 | deep link owner banner | — | IT-070 | E2E-018 |
| US-009.EC-3 | empty state names profile | — | — | E2E-019 |
| US-009.EC-4 | 100× volume no cross-effect | — | IT-016 | — |
| US-009.EC-5 | loop output only via owned run | — | IT-017 | — |
| US-010 | switch refilters everything | — | IT-023 | E2E-013 |
| US-010.AC-2 | one-sentence boundary answer | — | — | E2E-013 |
| US-010.AC-4 | workspace list identical | — | IT-018 | E2E-013 |
| US-010.EC-1 | quiet single state | — | — | E2E-013 |
| US-010.EC-2 | stream leaves view, session lives | — | IT-023 | — |
| US-010.EC-3 | archived mid-open fallback | UT-021 | — | — |
| US-010.EC-4 | second client independent view | — | — | E2E-020 |
| US-011 | aggregate labeled rows | — | IT-019 | E2E-015 |
| US-011.EC-1 | aggregate w/ only default | — | IT-019 | — |
| US-011.EC-2 | CLI --all-profiles labeled | UT-077 | — | E2E-005 |
| US-011.EC-3 | observe dashboards aggregate mode | — | IT-020 | — |
| US-012 | create in All → default + chip | — | — | E2E-015 |
| US-012.EC-1 | toast surfaces misfile | — | — | E2E-015 |
| US-012.EC-2 | palette/deep-link parity | — | — | E2E-015 |
| US-013 | usage per profile | UT-085 | IT-021 | E2E-021 |
| US-013.EC-1 | archived owner still counted | — | IT-021 | — |
| US-013.EC-2 | pre-profiles usage under default | — | IT-002 | — |
| US-014 | per-workspace remembered profile | UT-018 | IT-027 | E2E-013 |
| US-014.EC-1 | archived remembered → default | UT-020 | IT-029 | — |
| US-014.EC-2 | re-registered ws starts fresh | — | IT-028 | — |
| US-014.EC-3 | aggregate never restored | UT-019 | — | E2E-015 |
| US-015 | CLI precedence chain | UT-022..024 | — | E2E-004 |
| US-015.EC-1 | missing/archived --profile error | UT-021 | — | E2E-002 |
| US-015.EC-2 | env-only terminals don't touch memory | UT-023 | IT-027 | — |
| US-015.EC-3 | reserved words on acting cmds | UT-004 | — | — |
| US-015.EC-4 | aggregate rows carry owner field | UT-077 | — | E2E-005 |
| US-016 | user layer everywhere | UT-043 | IT-040 | — |
| US-016.EC-1 | shadowing inspectable | UT-045 | IT-040 | — |
| US-016.EC-2 | empty user layer composes | UT-044 | — | — |
| US-017 | profile layer scoped | UT-043 | IT-040 | — |
| US-017.EC-1 | personal path inside repo rejected | UT-047 | — | — |
| US-017.EC-2 | rename moves personal folders | — | IT-034 | — |
| US-017.EC-3 | disk add → next refresh | — | IT-041 | — |
| US-018 | repo base + per-name folders | UT-046 | IT-041 | — |
| US-018.EC-1 | dormant + create hint | UT-046 | IT-041 | E2E-022 |
| US-018.EC-2 | invalid entries reported | — | IT-041 | — |
| US-018.EC-3 | branch switch follows tree | — | IT-041 | — |
| US-019 | team adoption via hint | — | IT-042 | E2E-022 |
| US-019.EC-1 | existing name binds, no seeding | UT-059 | IT-042 | — |
| US-019.EC-2 | repo folder rename re-points hint | — | IT-042 | — |
| US-019.EC-3 | two repos, one profile name | — | IT-042 | — |
| US-020 | config layers + write targets | UT-025..030 | IT-043 | E2E-006 |
| US-020.EC-1 | persona defaults per profile | UT-028 | IT-043 | — |
| US-020.EC-2 | orphan profile config diagnostic | UT-031 | — | — |
| US-020.EC-3 | concurrent layer writes both persist | — | IT-044 | — |
| US-020.EC-4 | hooks layered; sandboxes user-only | UT-032 | IT-045 | — |
| US-020.EC-5 | one apply timeline names layers | — | IT-046 | — |
| US-021 | credential override per provider | UT-035..037 | IT-047 | E2E-007 |
| US-021.EC-1 | env ref refused for profile scope | UT-037 | — | E2E-007 |
| US-021.EC-2 | removal warning + fallback | — | IT-048 | E2E-007 |
| US-021.EC-3 | native logins shared, stated | — | — | E2E-007 |
| US-021.EC-4 | archived overrides dormant | — | IT-038 | — |
| US-022 | placement visibility | UT-055 | IT-053 | — |
| US-022.EC-1 | placement w/o profile dormant+hint | UT-056 | IT-053 | E2E-023 |
| US-022.EC-2 | enablement AND placement | UT-055 | IT-053 | — |
| US-022.EC-3 | update re-places, no copies | — | IT-054 | — |
| US-023 | declared profiles auto-create | UT-057..060 | IT-049..052 | E2E-008 |
| US-023.EC-1 | existing name = bind never seed | UT-059 | IT-050 | — |
| US-023.EC-2 | no resurrect; fresh install recreates | UT-060 | IT-052 | — |
| US-023.EC-3 | update never mutates created | — | IT-051 | — |
| US-023.EC-4 | uninstall leaves profile | — | IT-052 | — |
| US-023.EC-5 | invalid declared name fails install | UT-053 | — | — |
| US-023.EC-6 | two extensions same name | — | IT-050 | — |
| US-024 | per-profile enablement | UT-061 | IT-053 | E2E-023 |
| US-024.EC-1 | disable in own declared profile | — | IT-053 | — |
| US-024.EC-2 | dev links compose | — | IT-055 | — |
| US-024.EC-3 | disabled everywhere = inert | UT-061 | — | — |
| US-024.AC-4 | secret bindings per profile | UT-066 | IT-055 | — |
| US-025 | preset enablement per profile | UT-062 | IT-056 | E2E-026 |
| US-025.AC-3 | preset enablement surface parity | UT-090 | IT-071 | E2E-026 |
| US-025.EC-1 | archived pauses deliveries | — | IT-078 | — |
| US-026 | desktops per profile | — | IT-057 | E2E-024 |
| US-026.EC-1 | archived layouts return | — | IT-057 | — |
| US-026.EC-2 | no cross-contamination | — | IT-057 | E2E-024 |
| US-027 | memory tier per profile | UT-069..071 | IT-058 | — |
| US-027.EC-1 | pre-profiles memory → default | — | IT-058 | — |
| US-027.EC-2 | consolidation respects boundary | — | IT-059 | — |
| US-028 | immutable session binding | UT-072..074 | IT-066 | E2E-009 |
| US-028.EC-1 | acting as another profile rejected | UT-074, UT-091 | IT-066 | — |
| US-028.EC-2 | agent aggregate reads labeled | — | IT-019 | — |
| US-029 | cross-profile conversation | — | IT-060 | — |
| US-029.EC-1 | conversation owned by creating side | — | IT-060 | — |
| US-029.EC-2 | isolation request → not supported | UT-075 | — | — |
| US-030 | Global as view (phase 0) | UT-048 | IT-003..005 | E2E-010 |
| US-030.EC-1 | `~/` row gone; work → no-workspace | — | IT-003 | — |
| US-030.EC-2 | axes compose | — | IT-019 | E2E-015 |
| US-030.EC-3 | zero workspaces fresh start | — | IT-005 | — |
| US-031 | catalog filtered server-side | UT-082 | IT-023..025 | E2E-011 |
| US-031.EC-1 | replay obeys filter | — | IT-024 | — |
| US-031.EC-2 | aggregate streams explicit+labeled | — | IT-025 | — |
| US-032 | local-only management | — | IT-061 | — |
| US-032.EC-1 | remote stale ref consistent | — | IT-061 | — |
| US-032.EC-2 | no remote third mode | — | IT-061 | — |
| US-033 | list + selection inspection | UT-076 | IT-062 | E2E-004 |
| US-033.EC-1 | archived remembered → default+reason | UT-020 | — | — |
| US-033.EC-2 | never-visited → default+source | UT-018 | — | — |
| US-033.EC-3 | single default list, no ceremony | UT-016 | — | — |
| Migrations 00078/00079 + memory stream | disposition, backfill, rebuilds, ref rewrites, crash recovery | — | IT-001..010, IT-072, IT-073 | — |
| Enforcement (ADR-015) | two modes, fail-closed, full matrix | UT-080..082 | IT-011..026, IT-069, IT-070 | — |
| Grant ceilings (D24) | closures + pairwise meets pinned; scalar rank absent | UT-049..052 | — | — |
| CLI output contract | frames, formats, errors | UT-076..081 | — | E2E-002 |
| API parity + defaults | routes on both listeners | — | IT-063, IT-064 | — |
| Lifecycle protocol (ADR-012 / prepare-apply-finalize) | plans, revisions, recovery | UT-086 | IT-034, IT-072 | — |
| Scope rename (ADR-013) | `global`→`user` sweep | UT-033..034 | IT-006 | — |

## Unit Tests

### Profile domain (`internal/profile` — Part II Core Interfaces)

Canonical suite: `internal/profile` (new package, new suite — no existing owner).

- **UT-001** (happy): `Manager.Create` with `{name:"marketing", color:"#FF7F3A", icon:"megaphone"}` → Profile persisted, state `active`, ULID id, `CreatedAt` set.
- **UT-002** (error): `Create` with `"Marketing"`, `"mkt space"`, `"-x"`, 33-char name → `profile_name_invalid` naming the grammar.
- **UT-003** (error): `Create`/`Rename` to a name held by an active profile and by an archived profile → `profile_name_taken` naming holder + state.
- **UT-004** (error): `Create`/`--profile` value `default|all|global` → `profile_name_reserved` / rejection on acting commands.
- **UT-005** (state): `Create` without color/icon → auto-assigned starter identity, editable via `UpdateIdentity`.
- **UT-006** (happy): `UpdateIdentity` sets emoji, clears icon (mutual exclusion honored).
- **UT-007** (error): `UpdateIdentity` with `color:"red"`/`"#ggg"` → validation error; stored color unchanged.
- **UT-008** (state): `UpdateIdentity` on archived profile succeeds.
- **UT-009** (happy): `Rename("dev","eng")` result lists machine-folder rename performed, repo candidates, dormant placements; profile row keeps id.
- **UT-010** (error): `Rename`/`Archive`/`Delete` on `default` → `profile_permanent`.
- **UT-011** (state): `RenameOptions{Repos:all|none}` control repo offers in `RenameResult`.
- **UT-012** (happy): `Archive` on work-owning profile → state `archived`, `ArchivedAt` set, result lists paused automations.
- **UT-013** (error): `Archive` with running sessions → `profile_sessions_running` naming sessions.
- **UT-014** (idempotency): second `Archive`/`Unarchive` → no-op notice, no error.
- **UT-015** (happy+error): `Delete` on empty profile → removed + the **total** enumeration `{agents, skills, loops, mcp_servers, config_keys, credential_overrides, memory_entries, desktop_partitions}`; on work-owning → `profile_owns_work` with archive action.
- **UT-016** (boundary): `List` with only `default` → one row, no error; the `current` decoration is asserted at the surface layer (CLI/API cases), never in the domain result.
- **UT-093** (state): availability predicate — a profile with any non-`done` lifecycle op is unavailable (`profile_unavailable`) to resolver/selection/creation; `done` restores availability.
- **UT-017** (happy): each creation boundary's **own canonical suite** asserts the stamp value on insert — session upsert in the session store suite, task insert in the task service suite, loop-run insert in the loop coordinator suite, automation job/trigger in the automation suite. UT-017 is the shared ID for those per-suite assertions; no centralized cross-domain stamping test exists.
- **UT-086** (error): any plan-backed mutation with an outdated `PlanRevision` → `profile_plan_stale`; nothing commits (prepare is read-only).
- **UT-088** (boundary): `profiles` symbol constraint — exactly one of icon/emoji set; both-empty input auto-assigns before insert; both-set rejected.
- **UT-089** (happy): `work_items` contract — counts rows across the 17 stamped roots only; descendants excluded; worktrees included; same number feeds list, plans, Settings, labels.

### Archive/claim boundary (canonical suite: `internal/task` service tests)

- **UT-087** (state): the single claim query's eligibility predicate — a queued run whose owning profile is `archived` is never returned; flipping the profile back to `active` makes it claimable; `ClaimNextRun` signature unchanged.

### Resolver + selection (`internal/profile/resolve.go`, `selection.go`)

Canonical suites: `internal/profile` for the resolver/selection units; CLI-visible behavior joins `internal/cli/workspace_test.go` + `session_test.go`.

- **UT-018** (happy): chain — flag beats env beats remembered beats default; never-visited workspace → `{default, source:"default"}`; remembered hit → `source:"remembered"`.
- **UT-019** (state): `SelectionStore.Put` rejects aggregate/pseudo values; only real profile ids persist.
- **UT-020** (state): remembered entry pointing at archived/deleted profile → resolver returns default with `Note = archived_remembered_fallback`; a never-visited lens returns `Note = no_remembered_choice`; `SweepProfile` removes rows.
- **UT-021** (error): explicit flag/env naming missing profile → `profile_not_found`; archived → `profile_archived`; no silent fallback.
- **UT-022** (happy): `--profile` recorded on command context with source taxonomy mirroring workspace resolution.
- **UT-023** (state): env-selection resolution does not write the selection store.
- **UT-024** (error): `--profile x --all-profiles` → `profile_selection_conflict`.
- **UT-091** (error): resolver with `SessionProfileID` set and a differing Flag/Env on an acting path → `profile_session_conflict`; matching values resolve with source `session`.

### Config (`internal/config` overlay/write; `internal/settings` scopes)

Canonical suites: existing `internal/config` load/persistence suites and `internal/settings` suites — extended, not duplicated.

- **UT-025** (happy): `loadWithHome` merge order with 4 files — most specific wins per key; effective source named.
- **UT-026** (happy): `resolveWriteTarget` arms — scope `user`→user file; `profile`→active profile file; `workspace`→ws file.
- **UT-027** (state): context-owned default — active `default` → user file; active `marketing` → profile file.
- **UT-028** (happy): `defaults.agent|provider|sandbox` settable on profile layer (post-`general`-split section).
- **UT-029** (error): denylisted keys (`http.port`, `daemon.socket`, `gateway.*`, `shell.*`, `log.*`, `sandboxes.*`) on profile layer → `profile_config_key_denied` with allowed prefixes.
- **UT-030** (happy): write to layer overridden by more specific one → `OkOverridden`-style result naming winning layer.
- **UT-031** (error): profile config file with no matching profile → diagnostic, zero keys applied.
- **UT-032** (state): hook defined in profile layer resolves only under that profile; most-specific hook wins.
- **UT-092** (happy+error): MCP sidecar layers — `profiles/<p>/mcp.json` (user and project sides) merge in their layer's slot with five-layer precedence; writes target the context-owned layer; an orphan profile `mcp.json` (no matching profile) files a diagnostic and applies nothing.
- **UT-033** (state): two distinct enum assertions — config `WriteScope` accepts exactly `user|profile|workspace`; `settings.ScopeKind` accepts exactly `user|profile|workspace|agent` (the shipped `agent` scope is preserved); **both** reject only the obsolete `global` (ADR-013).
- **UT-034** (state): memory scope enum accepts `profile|workspace|agent`; `global` rejected.

### Vault + credentials (`internal/vault`, `internal/providers`)

Canonical suites: existing `internal/vault` ref suites and `internal/providers` cache suites.

- **UT-035** (happy): `vault:profiles/marketing/providers/openai/api_key` parses; namespace registered in pattern + map.
- **UT-036** (error): profile ref access from another profile's scope → containment rejection (owner-prefix rule).
- **UT-037** (error): `env:` ref for profile scope → `profile_secret_env_forbidden` at validation.
- **UT-038** (happy): provider key resolution — override present → profile ref; absent → user slot; source reported for `provider inspect`.
- **UT-039** (state): rename rewrite list — all `vault:profiles/dev/…` refs enumerated for `eng` rewrite, nothing outside prefix.
- **UT-040** (happy): `PreStartScope{ProfileID}` changes cache key; same provider+workspace, different profile → distinct keys.
- **UT-041** (state): `bridgesdk.InstanceCache` key includes profile; no cross-profile hit.
- **UT-042** (boundary): two identities, two assertions — the profile **name** is a valid vault ref segment; the stable profile **id** (zero-ULID form included) is the `@pf-<id>` extension-data segment (rename-stable); the name is never used for extension data.

### Resource layers (`internal/config/agent.go`, `internal/skills`, `internal/resources`)

Canonical suites: existing discovery/scanner suites in `internal/config` + `internal/workspace`, `internal/skills` registry suites, `internal/resources` type suites, `internal/extension` capability-policy suites (D24 pins).

- **UT-043** (happy): discovery roots order = project-profile → project base → profile → user (+ existing additional/bundled slots); winner per name matches order.
- **UT-044** (boundary): empty layers compose (no user resources; no repo folders) — remaining layers unaffected.
- **UT-045** (happy): shadow evidence lists winner layer + shadowed entries for `agent list` LAYER/SHADOWS columns.
- **UT-046** (state): per-name repo folder binds only when active profile name matches; mismatch → dormant with hint payload; rename-to-match wakes.
- **UT-047** (error): personal resource root pointing inside a registered workspace → rejected with reason.
- **UT-048** (state): user-layer resources visible under any workspace lens (post-D3), not only Global.
- **UT-049** (state): `ResourceScopeKind` accepts `user|workspace|profile|workspace_profile`; `Validate` binds scope_id rules (`user`⇒empty; `workspace_profile`⇒`<ws>@pf:<name>` segment).
- **UT-050** (state): scope-ceiling downward closures pinned for all four values — `user → {user, workspace, profile, workspace_profile}`; `workspace → {workspace, workspace_profile}`; `profile → {profile, workspace_profile}`; `workspace_profile → {workspace_profile}` (lattice, D24).
- **UT-051** (state): pairwise ceiling **meets** pinned — `meet(workspace, profile) = workspace_profile`; `meet(user, X) = X`; `meet(X, X) = X`; `meet(workspace|profile, workspace_profile) = workspace_profile` — proving no combination ever widens an axis (scalar minimums forbidden).
- **UT-052** (state): precedence rank function decoupled from enum iota — appending a source does not change existing ranks.

### Extension (`internal/extension`)

Canonical suites: existing manifest/validation/install suites in `internal/extension` (+ `internal/extensiontest` harness).

- **UT-053** (error): manifest with invalid/reserved declared profile name, or placement naming undeclared+nonexistent profile grammar violation → validation failure at build/install.
- **UT-054** (happy): manifest normalize — `[[profiles]]` block + per-resource `profile` keys parsed into typed config.
- **UT-055** (state): effective visibility = enabled(P) AND (unplaced OR placed-in-P-name) — truth table across 4 combinations.
- **UT-056** (state): placement naming absent profile → dormant entry with create hint (no resource published).
- **UT-057** (happy): install plan for declared profile — create + seed defaults + credential asks + enable, operator's active profile untouched.
- **UT-058** (happy): install summary payload names profiles to create + credential needs (US-023.AC-2).
- **UT-059** (state): declared name already exists → bind only; seed/defaults/identity not applied.
- **UT-060** (state): marker semantics — create-once per (instance,name); boot/enable/update/repair no-op; uninstall removes markers; fresh install = new instance may create.
- **UT-061** (state): enablement rows only for exceptions; absent row = enabled; disable in P hides resources in P only.
- **UT-062** (state): preset enablement per profile; absent row = **enabled** (uniform default-on; rows exist only for disables).

### Tools, identity, misc

Canonical suites: `internal/tools`, `internal/agentidentity`, `internal/memory`, `internal/notifications`, `internal/cli` format suites — each case in its owner.

- **UT-090** (state): preset enablement resolution — absent row = enabled (default-on); per-profile disable affects only that profile (notifications suite).

- **UT-066** (state): extension secret binding resolution — user binding default, profile override wins in owning profile only.
- **UT-069** (happy): `dirForScope(profile)` → `profiles/<p>/memory`; workspace/agent unchanged.
- **UT-070** (state): recall query scoped by profile tier — other profile's entries invisible.
- **UT-071** (boundary): legacy `~/.compozy/memory` content resolves under `profiles/default/memory` post-move.
- **UT-072** (happy): `tools.Scope.ProfileID` threads into native tool visibility projections.
- **UT-073** (happy): `compozy__profile_list` / `compozy__profile_current` return shapes from `_dx.md`; `source:"session"` inside sessions.
- **UT-074** (error): acting command with caller-supplied profile ≠ session binding → deterministic rejection (D9).
- **UT-075** (error): profile-restricted network delivery request → `not supported` (future scope).
- **UT-076** (happy): `profile list -o json` / `current -o json` payloads exactly as `_dx.md`.
- **UT-077** (happy): `--all-profiles` JSONL — resolution frame + `profile_name` on every row.
- **UT-078** (happy): profile-resolution JSONL frame emitted on empty lists (mirrors workspace frame).
- **UT-079** (error): every `_dx.md` error code maps through `marshalStructuredExecutionError` to `{"error":{code,message,action}}`.
- **UT-080** (state): store list methods require an explicit valid `ReadScope`; **both** invalid states — empty ProfileID with AllProfiles=false, and ProfileID set with AllProfiles=true — fail `ReadScope.Validate` before any query runs (fail-closed, zero rows).
- **UT-081** (state): aggregate row assembly joins owner identity (name/color/icon) incl. archived owners.
- **UT-082** (state): catalog-stream subscriber predicate — workspace+profile axes; `all_profiles` labeled; no-filter subscribe impossible by construction.
- **UT-085** (happy): usage accumulator writes `(day, profile_id, workspace_id, agent_name)`; attribution independent of credential used.

## Integration Tests

### Migration `00078` + phase-0 rewrite

- **IT-001**: fresh DB → schema equals declarative definitions after `00078` + `00079` + the memory-stream migration (equivalence suite) incl. `profiles`, `profile_selections`, join tables, mutes, stamps, triggers.
- **IT-002**: reopen pre-profiles DB → backfill assigns `default` to all 17 roots' rows; counts preserved; usage history intact under default.
- **IT-003**: phase-0 disposition equivalence (positive) — one seeded fixture with rows in **every rewrite/drop family** (spec table) survives `00078`: each family lands in its **labeled shape** (shape 1 = NULL for plain FKs; shape 2 = `sessions` only, `''` + `scope`; shape 3 = `''` alone with rebuilt CHECKs — UNIQUEs still dedup, composite FKs still enforce), drop-row families removed, counts/identities/relationships preserved; the `~/` row is gone and nothing cascaded.
- **IT-004**: `token_usage_daily` + `notification_cursors` PK rebuilds — rows preserved, new identity queryable, old identity gone.
- **IT-005**: migration `00079` seeds `default` (fixed zero ULID) on fresh and upgrade paths **before** any backfill; boot's ensure only verifies the row and creates its directory layout idempotently (second boot no-op); zero-workspace fresh start serves the Global lens.
- **IT-006**: stored scope values rewritten `global`→`user` (resource_records, MCP auth/registration rows, settings payload round-trip) and memory `global`→`profile` (its own stream migration); stored `vault:mcp/global/…` / `vault:extensions/global/…` refs rewritten to `…/user/…`; no reader accepts old values.
- **IT-007**: stamp immutability — `UPDATE … SET profile_id` on each root → ABORT.
- **IT-008**: profile FK RESTRICT — deleting a profile row with owned work fails at SQL layer (service gate tested separately).
- **IT-009**: workspace-delete trigger cascades `profile_selections` rows.
- **IT-010**: memory dir move — existing global-tier files land under `profiles/default/memory` and recall finds them.

### Enforcement sweep (fail-closed, two modes)

- **IT-011**: sessions list/get scoped — profile A sees only A across HTTP+UDS; absent param = default; unknown param → 404 body `profile_not_found`.
- **IT-012**: tasks + loops + automations listings scoped identically (one fixture, three surfaces).
- **IT-013**: attention/approval counters scoped; badges match row counts.
- **IT-014**: bridge-delivered inbound work owned by instance's profile (mock bridge fixture).
- **IT-015**: worktrees listed under every profile with owner tag (ruled exception).
- **IT-016**: 10k rows in profile A → profile B listings latency/pagination unaffected; cursor pages stable.
- **IT-017**: loop output fetch via owned run OK; direct ref from other profile's scope → not found; no unscoped listing route exists.
- **IT-018**: workspace catalog identical across profiles (list + picker payload).
- **IT-019**: aggregate mode — labeled rows incl. archived owner; composes with workspace filter and Global lens; agent-initiated aggregate read allowed + labeled.
- **IT-020**: observe overview/dashboard/inbox in labeled-aggregate mode; per-profile breakdown rows sum to totals.
- **IT-021**: usage endpoints scoped + aggregate; attribution follows owner with mixed credentials (override in one profile).
- **IT-022**: `situation` `/agent/context` bundle contains only session-profile data (leak probe with foreign-profile fixtures).

### Streams & cursors

- **IT-023**: catalog stream live filter — events from other profiles/workspaces never reach a scoped subscriber (wire-level assertion); profile switch bumps generation, old stream stops writing.
- **IT-024**: reconnect replay obeys the same filter (`after_seq` replay across profile boundary yields nothing foreign).
- **IT-025**: `all_profiles=true` stream labeled; only explicit widening path.
- **IT-026**: listcursor fingerprint includes profile predicate — cursor minted in A replayed in B → invalid/empty, never B's rows.

### Selection

- **IT-027**: `profile use` under workspace my-saas persists row; CLI in second terminal resolves it; env selection does not persist.
- **IT-028**: unregister + re-register workspace → selection starts at default.
- **IT-029**: archive **keeps** the selection row — next entry resolves `default` with `Note = archived_remembered_fallback`; unarchive restores the original selection untouched; delete sweeps the row — next entry resolves `default` with `Note = no_remembered_choice` (US-014.EC-1 / US-006.EC-5 / US-033.EC-1..2).
- **IT-030**: Global-lens slot independent of workspace slots.

### Lifecycle flows

- **IT-031**: create → active immediately; identity persisted; visible via list on both listeners with identical fields.
- **IT-032**: concurrent create same name → exactly one 201, one 409 `profile_name_taken`; delete-vs-create race → delete fails `profile_owns_work`.
- **IT-033**: concurrent identity edits → last write wins; both clients converge on GET.
- **IT-034**: rename via the lifecycle protocol — apply commits catalog + `vault:profiles/*` ref rewrites in one immediate transaction after plan-revision validation (stale plan → `profile_plan_stale`, nothing commits); finalize renames machine folders idempotently; **selections (id-keyed) remain untouched and valid**; running session keeps working under the new name.
- **IT-035**: rename repo handling — accepted repos get folder rename (working-tree change); declined/unreachable → dormant hints listed; rename-back wakes bindings.
- **IT-036**: archive — blocked with running session; then stops → succeeds, automations paused (rows show paused reason), no new automation work created; archived owner's rows visible only in aggregate. Existing notification-delivery pause behavior is owned by IT-078.
- **IT-037**: unarchive restores switcher visibility + resources/config/creds intact; paused automations require explicit reactivation; all-archived degrade leaves default functional.
- **IT-038**: delete — enumeration matches actual removals **exactly and totally** (personal folders incl. loops, MCP sidecar server entries, config layer file, credential refs + requirement rows, profile-tier memory rows + directory, clientstate desktop partitions); plan payload, CLI output, and DELETE result agree field-for-field; nothing profile-owned survives in any store; name freed.

### Resource layers

- **IT-040**: layer composition end-to-end — user-layer and profile-layer resources compose additively; a profile-layer resource is present in every workspace under its owning profile and absent under others; a same-name conflict resolves to the most specific layer, and `agent list` reports the LAYER/SHADOWS columns (winner + shadowed entries) identically on both transports.
- **IT-041**: repo layers live — project-base resources visible under every profile in that workspace; a `profiles/<name>/` folder binds only while the active profile's name matches; content naming an absent profile is dormant with the create-hint payload on the workspace surface; a resource added on disk appears on the next catalog refresh in its owning scope only; invalid entries are reported while valid ones load; a branch switch refreshes the catalog to the working tree with no stale ghosts.
- **IT-042**: team adoption — registering a repo with committed `profiles/dev` + `profiles/marketing` content surfaces the hint naming both; creating one name from the hint binds its content immediately and the hint drops that name; a pre-existing same-name profile binds with no seeding; a repo folder rename re-points the hint to the new name; two registered repos declaring the same name bind one profile, each only inside its own workspace.

### Config & credentials

- **IT-043**: end-to-end write targeting — under default → user file; under marketing → profile file; `--scope` overrides; effective value + winning layer via inspect; UI settings write path hits same targets.
- **IT-044**: concurrent writes to same key in user and profile layers — both files correct; effective follows precedence.
- **IT-045**: profile-layer hook fires only under owning profile (real hook dispatch); sandbox section in profile file → load-time validation error.
- **IT-046**: `config_apply_records` single timeline; entries name layer/file for user and profile writes.
- **IT-047**: provider override resolution in real run path — marketing run uses override key, default run uses user key (acpmock asserts injected credential), attribution per owner either way.
- **IT-048**: `secret rm` with owned running work → warning payload; in-flight completes on resolved credential; next run resolves user key.

### Extensions

- **IT-049**: install with declared profile — profile created once, seeded, extension enabled there, active profile unchanged, summary lists creation; `needs_setup` is durable profile-owned state (`profile_credential_requirements`): visible in list/get projections, surviving daemon restart, extension update, **and uninstall**, cleared only by the canonical vault write; fail-closed use with plain error before setup.
- **IT-050**: declared name pre-exists → bind-only (identity untouched); two extensions declaring same name → first creates, second binds; independent markers.
- **IT-051**: update adding new declared profile creates only it; update changing seed of created profile → no mutation.
- **IT-052**: archive/delete declared profile → boot/re-enable/update/repair never recreate; uninstall removes markers + resources but leaves profile/work; fresh reinstall (new instance) recreates.
- **IT-053**: placement matrix live — 1 shared skill + 2 in `x` + 1 in `y`; visibility follows enablement AND placement; disable in `x` hides placed resources there.
- **IT-054**: update changing a resource's placement → new placement effective, old gone.
- **IT-055**: dev-linked extension composes with per-profile enablement; secret binding override per profile resolves scoped.
- **IT-056**: preset enablement per profile changes delivery routing; library shared.

### Per-profile state, memory, network, identity

- **IT-057**: window-manager snapshots keyed `(workspace, profile)` — arrangements isolated, restored on switch, archived retained, new profile clean.
- **IT-058**: memory recall isolation across profiles at former-global tier; workspace tier shared; pre-profiles memory under default.
- **IT-059**: consolidation runs never merge two profiles' tiers (fixture with both).
- **IT-060**: network conversation dev↔marketing delivered (regression: tag ≠ block); conversation rows owned by creating side; peer registry unchanged by profile ops.
- **IT-061**: remote/paired tier — **every profile-state write** (lifecycle mutations, `PUT /api/profiles/selection`, enablement writes) → 403 `profile_remote_management_forbidden`; reads scoped; stale profile ref → consistent empty/fallback; no third read mode.
- **IT-062**: selection/current inspection — `GET /api/profiles` + selection routes on both listeners byte-parity; `source` values correct per fixture (flag/env/remembered/session/default).
- **IT-063**: HTTP/UDS route parity test for the profile route set (first parity guard — path-map correction 13).
- **IT-064**: API defaults/conflicts — absent `profile` = default; `profile`+`all_profiles` → 400 `profile_selection_conflict`; archived profile param → 409 `profile_archived`.
- **IT-065**: (withdrawn — `make codegen-check` owns generated-artifact drift; behavioral transport parity lives in IT-063.)
- **IT-066**: agent-identity derivation — native tool + CLI-in-session resolve the session's profile (source `session`); a differing `--profile`/`COMPOZY_PROFILE` on an acting path fails `profile_session_conflict` with the same status/body contract as UT-091/UT-074, and the session binding is proven unchanged afterward (D9).

### Round-1 additions (archive/claim, plans, presets, protocol, migrations)

- **IT-067**: archive-vs-claim race — `BEGIN IMMEDIATE` archive and a concurrent claim serialize; whichever commits second observes the other (claim after archive returns nothing for that owner; archive after claim aborts on the leased run); no archived-owner claim ever succeeds post-commit.
- **IT-068**: queued-run freeze — queued runs owned by an archived profile are skipped by claims while frozen, counted in the archive result, and become claimable after unarchive (single queue, single claimer throughout).
- **IT-069**: network (channels/threads/direct/work/timeline), bridge (instances/routes/deliveries/subscriptions), and audit (`permission_log`, `network_audit_log`) listings scoped + labeled-aggregate per the enforcement matrix; fail-closed on indeterminate scope.
- **IT-070**: labeled aggregate-by-id — scoped single-get of a foreign item → 404; `all_profiles=true` single-get → owner-labeled resource (deep-link contract), both transports.
- **IT-071**: preset enablement — CLI verbs, routes, and Settings write path hit the same store; effective-state list per profile correct; parity across HTTP/UDS.
- **IT-072**: lifecycle crash recovery — kill between apply and finalize for create/rename/delete; boot reconciliation converges directories to the catalog deterministically; re-run is idempotent.
- **IT-073**: memory-stream migration — scope values + indexes rewritten; the durable `memory_maintenance_ops` row drives the move: crash-interrupted rename re-runs to convergence (idempotency guard), profile-tier reads refuse while pending, old path absent and row `done` afterward.
- **IT-074**: memory read-path isolation — FTS/recall/catalog queries in profile B never return A's entries even with identical queries warmed in A (cache identity includes profile); aggregate memory read requests are refused with a typed error; native-tool + situation memory projections scoped.
- **IT-075**: phase-0 assert-empty negatives — one fixture per assert-empty family (worktrees of `~/`): migration `00078` aborts loudly naming the family; nothing is partially applied.
- **IT-076**: archived-owner creation guard — direct insert attempts at each of the 17 roots for an archived profile abort with `profile_archived` (per-root trigger), covering manual create, automation trigger, bridge delivery, and spawn boundaries in the same write transaction.
- **IT-077**: availability gate — while a lifecycle op is `finalizing` (or `failed`), the profile is rejected everywhere with `profile_unavailable` (selection, exec/work creation via the trigger, discovery, provider prestart); boot reconciles the journal before the API accepts traffic; `profile ops` lists the op and `retry` converges a failed one.
- **IT-078**: notifications vs archive (owner-active **permit** protocol) — barriers at three points: (a) archive commits before permit acquisition → dispatch refused, nothing sent, pause reason inspectable; (b) permit held through an external send → archive attempt fails with the public `409 profile_deliveries_in_flight` payload (asserted body, retryable) until ack + cursor advance delete the permit row; (c) crash with the marker set → recovery replays by delivery id (no duplicate) and advances; unarchive resumes from the preserved cursor with no double delivery.
- **IT-079**: canonical event coverage — the existing observability coverage suite extends to every profile lifecycle path (`created/renamed/identity_updated/archived/unarchived/deleted`, `selection_changed`, `extension.profile_created`, `extension.enablement_changed`, `notification_preset.enablement_changed`, `lifecycle_op_recovered/failed`, `plan_stale`); no standalone duplicate event test.
- **IT-080**: name/path reservations — rename-vs-same-name-create and delete-vs-same-name-create races (barrier-controlled): the second op fails `profile_name_taken` naming the pending operation; boot recovery of a `finalizing`/`failed` op with a would-be replacement profile present never touches the replacement's paths.
- **IT-081**: enablement hard cut — after `00079`, `extensions.enabled` / `notification_presets.enabled` columns are gone (equivalence suite), prior disabled state appears as `default`-profile exception rows, and HTTP/UDS/CLI expose only the per-profile authority (old fields absent from payloads).
- **IT-082**: `extension_env_bindings` rebuild — new `(extension_name, profile_id'', workspace_id'', env_name)` identity in place (equivalence suite); the **total resolution order** is pinned with all four rows present and every pairwise conflict: `(profile,ws)` → `(profile,'')` → `('',ws)` → `('','')` (profile axis outranks workspace axis; absence falls through; no row = unbound); cleanup triggers fire on extension, workspace, and profile deletion; rename leaves bindings untouched (id-keyed).
- **IT-084**: memory durable ownership — migration adds `profile_id` to catalog/consolidation/decision/event/recall identities + FTS indexes; former-global rows land on the default zero-ULID; two profiles writing the same `(workspace, scope, agent, tier, type, slug)` coexist without collision and never cross-read (extends IT-058/074 fixtures).
- **IT-085**: declared-seed crash barrier — kill between apply and the seed-backed config write: recovery reproduces the exact persona defaults + credential asks from the journaled seed snapshot (never from the manifest, which the test mutates post-crash to prove it); `declaration_digest` matches the accepted declaration; after the op reaches `done`, the requirement rows (not the op snapshot) answer needs-setup — lifecycle-op retention cleanup never erases the signal.
- **IT-083**: session creation witness — otherwise-identical sessions created in two profiles produce **different** creation digests; reseed and goal-binding paths thread the profile id; `loop_runs.origin_creation_profile_ref` semantics unchanged (extends the canonical session-creation identity suite).

## End-to-End Tests

### Runtime (Go harness, CLI transcripts from `_dx.md`)

- **E2E-001**: golden path verbatim — create marketing → exec → `session list` scoped → `--all-profiles` labeled → `use default` → `current` shows remembered source.
- **E2E-002**: error surface sweep — invalid name, reserved name, unknown `--profile`, archived selection, `--profile`+`--all-profiles`; each exits 1 with exact `{"error":{code,…}}` payload under `-o json`.
- **E2E-003**: lifecycle guardrails — archive-with-running-session message; delete-with-work offers archive; `default` permanence trio.
- **E2E-004**: two terminals — `COMPOZY_PROFILE=marketing` vs flag `dev` concurrently; `profile current` sources correct; remembered untouched by env.
- **E2E-005**: `session list --all-profiles -o jsonl` — resolution frame + owner fields byte-shape.
- **E2E-006**: config trio — save to profile; `--scope user` overridden feedback; denylisted key error with `--scope user` guidance.
- **E2E-007**: credentials journey — `secret set` under marketing (vault ref echoed), `provider inspect` source lines, env-ref refusal, `secret rm` warning.
- **E2E-008**: extension install with declared profile (mock kit) — summary, creation, needs-setup line, `use growth` + secret set completes setup.
- **E2E-009**: in-session agent — `compozy__profile_current` returns session source; lifecycle mutation attempt from remote-tier token refused.
- **E2E-010**: phase-0 — `workspace add ~/` refused with reason; Global-view creation produces no-workspace work owned by active profile.
- **E2E-011**: catalog stream wire test — scoped subscriber receives zero foreign events during cross-profile activity burst.
- **E2E-012**: machine commands (`daemon status`, `doctor`) identical output with and without profile selection.

### Web (Playwright, `_uiux.md` journeys)

- **E2E-013**: S1 — quiet single state; create "marketing" via switcher (picker: icon tab, color); switcher becomes identity element; switch refilters listings; boundary sentence present; workspace picker list identical across profiles; return to workspace restores its profile.
- **E2E-014**: S4/S5 — Settings page list/create/edit identity (emoji tab swap, invalid hex inline error), archived list under disclosure.
- **E2E-015**: S2/S3/S11 — "All profiles" labeled rows (archived owner muted); destination chip visible on creation surfaces; owner toast names `default`; palette-creation parity; axes compose with Global toggle; leaving All restores last real profile.
- **E2E-016**: S6 — rename dialog: machine tier informational, repo offers pre-checked, declined repo shows dormant hint after.
- **E2E-017**: S7 — archive confirm lists paused automations; blocked state names running session; unarchive lists reactivation items.
- **E2E-018**: deep link to another profile's session → owner banner + one-tap switch; surrounding lists stay scoped.
- **E2E-019**: empty state names active profile ("No sessions in Marketing yet").
- **E2E-020**: two browser contexts — independent active profiles; only remembered choice shared after explicit switch.
- **E2E-021**: usage surface — scoped figures per profile; aggregate labeled breakdown.
- **E2E-022**: S9 — register repo with committed `profiles/dev` → hint; adopt creates + binds; hint drops name.
- **E2E-023**: S8 — extension detail placement matrix; per-profile enablement toggle; needs-setup badge until credentials filled; dormant placement hint.
- **E2E-024**: S12 — arrange windows in two profiles, switch back and forth → exact restoration each way; new profile starts clean.
- **E2E-025**: SymbolPicker keyboard-only journey (tabs, search, select, color) — a11y assertions (focus trap, labels).
- **E2E-026**: notification presets settings — per-profile enablement rows reflect the active profile; toggle persists; switching profile shows that profile's state (S13).
