# Test Specification: CompozyOS Merge (AGH → Compozy v0.3.0)

The canonical test contract for the `compozy-migration` program. It complements `_techspec.md`. There is no `_user_stories.md`; journeys derive from the brief's P0 list and migration-guide command map, as recorded in the TechSpec assumptions. Behavior sources are marked `P0-N` (a brief P0 item) or by component.

## Strategy

- Frameworks and harnesses: Go standard-library testing with `t.Run("Should …")` subtests and `t.Parallel` where process state is not mutated, table-driven matrices, and `-race`/`CGO_ENABLED=1`. Fakes stop at I/O boundaries: `acpmock` is the ACP subprocess boundary (including the reviewer/fixer agent sessions of `review-and-fix`), artifact writes use temporary trusted workspace roots plus containment checks, migrator fixtures use real-shaped v0.2.15 TOML homes, and unit-only catalog validation uses an in-memory `RuntimeCatalog`. Integration tests use the real model-catalog service with fixture sources.
- Execution: during iteration, run the scoped `go test -race ./internal/<pkg>/...` owner package; integration uses the `integration` tag; E2E uses `make test-e2e-runtime` (Go harness plus `acpmock`) and `make test-e2e-web` (Playwright). `make verify` is the completion gate. The release-PR dry-run owns IT-017 and is required release evidence, but is not folded into `make verify`.
- Conventions: API cases assert both status code and response body; injected clocks and IDs make time-dependent behavior deterministic; cases using `t.Setenv` stay serial. Rebrand sweeps are phase gates (`make verify`, `make codegen-check`, and task-scoped class grep commands), intentionally not prose/grep test suites. Behavior-bearing renamed surfaces retain their existing tests: reserved tool prefix, home/environment resolution, `ProtocolV0`, envelope keys, and real CLI/API/native-tool behavior. The currently discovered product-contract specs under `packages/site/lib/__tests__/` co-ship with their surface; no fragile count is pinned here. Markdown scope checks remain task verification commands, never standalone prose tests.
- Placement: every case below names a durable observable invariant and reuses its existing owner. Unit cases own pure resolution, parsing, validation, and serialization. Integration cases own persistence, reopen behavior, HTTP/UDS/CLI/native-tool parity, workspace boundaries, subprocess wiring, and filesystem containment. E2E owns user journeys. A static guide-parity command is allowed only because the two complete migration guides are a shipped product contract and no stronger owner exists.
- HTTP availability: the daemon HTTP API is always available on its configured listener. Deep-link coverage therefore verifies the configured host, port, and real `/loop-runs/<run_id>` route; it does not invent an unsupported "HTTP disabled" branch.

## Coverage Matrix

| Source | Behavior | Unit | Integration | E2E |
| --- | --- | --- | --- | --- |
| P0-2 / AGH-80 | Per-task runtime precedence, including complexity and node `params.runtime`, with field-level merge | UT-001–UT-009, UT-060–UT-061 | IT-001, IT-002 | E2E-001 |
| P0-2 | Resolved runtime is validated after effective layers resolve, before spawn; aliases canonicalize | UT-010–UT-013, UT-062 | IT-003, IT-006 | E2E-001 |
| P0-2 | Task frontmatter `runtime` and `type` parsing (`import_tasks`) | UT-014–UT-016 | IT-001 | E2E-001 |
| P0-2 | Binder applies the runtime triple while policy/auth remain authoritative | UT-017–UT-019 | IT-002, IT-004 | E2E-001 |
| P0-2 | Judge runtime uses defaults plus per-criterion values, never task rules | UT-020–UT-022 | IT-005 | — |
| P0-3 | `--runtime` grammar produces ordered per-run overrides | UT-023–UT-027 | IT-006 | E2E-001 |
| P0-2/3 | `resolved_runtime` survives store reopen and remains truthful across status, SSE, CLI, and native tools without workspace leakage | UT-028 | IT-007 | E2E-004 |
| Config lifecycle | Runtime defaults/rules, complexity precedence, and strict removal of `model_defaults`/`params.model` | UT-029–UT-032 | IT-008 | — |
| Config lifecycle / ADR-007 | Loop input defaults resolve only at effective dry-run/submission time, with typed errors and provenance | UT-063–UT-066 | IT-018 | E2E-001 |
| P0-1 | Review artifact deterministic byte shape, agent-authored issue intake, round numbering, trusted-workspace containment, and finalization | UT-033–UT-044 | IT-009–IT-012 | E2E-002 |
| P0-7 | Bundled dev-cycle skills are immutable, parseable, and isolated between workspaces | UT-045–UT-047 | IT-013 | — |
| P0-4 (deferred — task 14) | v0.2.15 config translation, exact review defaults, complete translate/drop reporting, backup, and idempotency | UT-048–UT-055 | IT-014 | E2E-003 |
| P0-6 | `loop run` emits a configured-host web deep link to the real run route | UT-056 | IT-015 | E2E-004 |
| Rebrand behavior | Home, environment, tool prefix, protocol value, and envelope keys resolve after the hard cut | UT-057–UT-059 | IT-016 | — |
| Phase 3 | Release identity and installer contract | — | IT-017 | — |
| Migration journey | A seeded delivery run completes with deep link (MVP); in-place legacy upgrade is deferred with task 14 | — | — | E2E-001; E2E-003 (deferred) |

Every component row includes error behavior. An empty E2E column means the named lower-layer owner is sufficient for that invariant.

## Unit Tests

### Runtime resolution engine (`internal/loop/runtime_resolve.go`; TechSpec: Core Interfaces)

- **UT-001** (happy): `ResolveItemRuntime` with only `Defaults{claude/opus/xhigh}` returns that triple with `source=default` for every field.
- **UT-002** (happy): frontmatter `{model: gpt-5.5}` over defaults `{codex/gpt-5.4/high}` resolves `{codex, gpt-5.5, high}`; only model reports `frontmatter`, while the other fields report `default`.
- **UT-003** (happy): a per-run `type=frontend → {provider: claude}` rule, frontmatter `{model: opus}`, and a config `type=frontend → {reasoning: high}` rule resolve `{claude, opus, high}` with `run`, `frontmatter`, and `config` provenance respectively.
- **UT-004** (ordering): for one task, specificity is deterministic: complexity defaults lose to type rules, type rules lose to task-id rules, and a later equally specific rule wins. The assertion is field-level, not whole-rule replacement.
- **UT-005** (ordering): two `type=frontend` rules of equal specificity resolve to the later rule.
- **UT-006** (happy): a per-run rule wins over frontmatter for the same field; frontmatter wins over config for that field. Pairwise assertions make each precedence relation explicit.
- **UT-007** (boundary): an item with `TaskID=""` and `TaskType=""` receives worker defaults only; matcher rules never match it; empty matchers are rejected by validation rather than becoming catch-alls.
- **UT-008** (state): unmatched fields stay empty in `ResolvedRuntime`; agent-definition fallback belongs exclusively to the binder.
- **UT-009** (happy): a `run_loop` child receives no parent per-run rules; its resolution input is built only from its own layers.
- **UT-060** (happy): rendered node `params.runtime` `{model: sonnet}` over `{claude/opus/high}` resolves `{claude, sonnet, high}` with `model.source=node` and default provenance for the other fields.
- **UT-061** (ordering/error): node runtime loses to every per-item layer—frontmatter, config, and per-run rules—and wins only over runtime defaults. The legacy scalar `params.model` is rejected at DSL parse/validation time; it is never silently translated into `params.runtime.model`.

### Runtime validation (`RuntimeCatalog`; TechSpec: Safety Invariant 3)

- **UT-010** (error): an unknown provider `flarp` makes `ValidateResolvedRuntime` return `runtime_validation` with `{field: provider, value: flarp, reason: unknown_provider}`.
- **UT-011** (authority boundary): the authoritative Cursor catalog rejects an unknown Cursor model with provider/model/task ID in the error; a provider without an authoritative model catalog accepts an otherwise valid non-empty model string rather than returning a false `unknown_model` result.
- **UT-012** (error): static `loop validate` accepts the canonical `none|minimal|low|medium|high|xhigh|max` reasoning vocabulary and rejects an empty matcher or any value outside it with a precise error. It does not claim to validate workspace- or run-dependent effective values; those are owned by effective dry-run/submission coverage.
- **UT-013** (boundary): an empty triple passes pre-agent validation because there is no explicit value to validate before agent resolution.
- **UT-062** (happy): a provider alias such as `z-ai` canonicalizes through `CanonicalProviderName` before catalog lookup; gateway slash model IDs such as `openrouter` plus `anthropic/claude-opus-4-7` pass; `resolved_runtime` exposes canonical IDs only.

### Task frontmatter parsing (`extensions/dev-cycle/import_tasks_parser.go`; TechSpec: Data Models)

- **UT-014** (happy): a `task_02.md` fixture with existing `complexity`, `type: frontend`, and a complete `runtime:` block populates `Type` and `Runtime`, exposed as `.item.type` and `.item.runtime`; strict known-field checking is scoped to nested `runtime` and does not reject unrelated valid frontmatter.
- **UT-015** (happy): absent `runtime:` produces `Runtime == nil`; absent `type` produces an empty string; both remain valid.
- **UT-016** (error): malformed `runtime:` such as `reasoning: 5` or an unknown `ide:` key fails import with a file- and field-scoped error, never a silent skip.

### Binder application (`internal/daemon`, `internal/config`; TechSpec: Core Interfaces, Invariants 4–5)

- **UT-017** (happy): `ResolveSessionAgentWithRuntime(agent, RuntimeOverrides{Provider:"codex"})` re-derives command/auth/home from Codex and clears the agent-definition model; `{Reasoning:"high"}` sets `ResolvedAgent.ReasoningEffort`.
- **UT-018** (error): a provider override that violates its policy, such as a missing bound-secret slot, fails at the policy gate and leaves engine state untouched.
- **UT-019** (state): `baseCreateOptions` copies the complete triple into `CreateOpts{Provider, Model, ReasoningEffort}`. Empty fields remain empty for the agent-definition path, while `AppliedRuntime` equals the fully applied triple including agent-definition fields.

### Judge runtime (`internal/loop/gate`; TechSpec: ADR-001 scope)

- **UT-020** (happy): criterion `runtime{model: opus}` overrides `runtime_defaults.judge.model`; unset criterion fields inherit judge defaults field by field.
- **UT-021** (happy): judge validation uses the same `RuntimeCatalog`; an invalid judge provider fails effective validation.
- **UT-022** (state): task-type, task-id, and complexity rules never apply to judges.

### `--runtime` flag parser (`internal/cli/loop_runtime_flag.go`; TechSpec: API Endpoints and Public Surfaces)

- **UT-023** (happy): `worker=claude/opus@xhigh` creates a complete `runtime_defaults.worker`; `judge=codex/gpt-5.5` leaves reasoning empty; bare model sugar `worker=opus` is equivalent to `worker=-/opus`.
- **UT-024** (happy): `type=frontend:claude/opus` and `id=task_03:codex/gpt-5.5-codex@high` produce one correctly keyed rule each.
- **UT-025** (happy): slash-safe model parsing and `-` skips work: `type=docs:-/gpt-5.5-mini` sets only model; `worker=openrouter/anthropic/claude-opus-4-7@high` keeps the model's internal slash; `worker=claude/-@high` sets provider and reasoning only.
- **UT-026** (error): missing `:` in a rule, unknown `foo=` selector, empty `type=x:`, duplicate selector collision, invalid reasoning suffix, and a match value containing `:` each return a distinct cited parse error.
- **UT-027** (idempotency): repeated flags preserve order without deduplication, so the later equal-specificity rule wins during resolution.

### Run-status contract (`internal/api/contract`; TechSpec: API Endpoints and Public Surfaces)

- **UT-028** (happy): the run-status payload serializes `resolved_runtime{provider,model,reasoning,source{…}}` per task from `AppliedRuntime`; agent-only values report `source=agent`, node values report `source=node`, and the typed contract remains the single projection consumed by store/API/SSE/CLI/native owners.

### Configuration layers (`internal/loop/config.go`; TechSpec: Data Models; API Endpoints and Public Surfaces)

- **UT-029** (happy): `runtime_defaults` and `runtime_rules` merge contract → global delivery/watch defaults → workspace/stored loop config → per-run overrides. Complexity defaults establish the least-specific task layer, followed by type then task-id rules; provenance survives each merge.
- **UT-030** (happy): `[loops.defaults.watch]` and delivery sections still route through `definitionLooksWatch`.
- **UT-031** (error): `model_defaults` in any supported config layer is rejected by strict decode with an error pointing to the migration guide.
- **UT-032** (state/error): `LoopGateCriterion.runtime` decodes, while legacy `LoopGateCriterion.model` and every legacy scalar `params.model` are rejected. No compatibility field, alias, or silent conversion remains.

### Review artifact writer (`extensions/dev-cycle/review_artifacts.go`; TechSpec: ADR-003, Invariants 6–9)

- **UT-033** (happy): `WriteReviewArtifacts` with two fixture `ReviewIssue`s writes `reviews-001/issue_001.md` and `issue_002.md` byte-identically to the v0.3-owned golden fixtures, including deterministic frontmatter field order, body headings, and `Decision: \`UNREVIEWED\``. The frontmatter carries no provider provenance keys (`provider`, provider reference, `review_hash`, source review ID/submitted-at do not exist in the contract); the writer, not the caller, owns `round`, `round_created_at`, and initial `status: pending`.
- **UT-034** (error): a request without `task_name` (or with an empty one) is rejected with a structured error naming the field; no `pr-<N>` derivation path exists and no directory is created.
- **UT-035** (boundary): an issue without `file` serializes `file: unknown` and batches under that same `unknown` filename value; an empty `author` serializes `author: unknown`. `title`, `body`, and `severity` are schema-required upstream, so the writer rejects a record missing them with a structured error instead of inventing legacy provider fallbacks. No `__unknown__` sentinel or omitted file field is introduced.
- **UT-036** (error): empty `Issues` creates no round directory and reports zero files; loop termination depends on the clean review result, not a writer side effect.
- **UT-037** (happy): existing `reviews-001` and `reviews-003` make the next round `004`; gaps are tolerated.
- **UT-038** (concurrency): two concurrent writers in one trusted task directory use exclusive directory creation: one wins the round and the other retries the next number, with no interleaved files.
- **UT-039** (security): a task name traversal, a task directory symlink escaping `.compozy/tasks`, or a workspace-root symlink resolving outside the trusted workspace is rejected before any write. The same task name in two trusted workspaces resolves only inside its caller workspace and cannot cross-read or cross-write.
- **UT-040** (history): rounds are append-only — re-invoking the writer with a new payload creates the next round and never mutates an earlier round's directory or files; artifacts from prior rounds remain byte-stable. No provenance-based dedupe exists (ADR-008): filtering repeat findings is the reviewer's job, not the writer's.
- **UT-041** (happy): `FinalizeReviewRound` changes `valid` and `invalid` files to `resolved` in place while preserving all other frontmatter bytes.
- **UT-042** (state): `pending` files stay untouched and contribute to `Pending`, so the loop continues.
- **UT-043** (idempotency): running finalize twice makes the second invocation a no-op with identical counts.
- **UT-044** (error): finalizing a nonexistent round returns a deterministic not-found error naming task and round.

### Skill bundling (`extensions/dev-cycle`; TechSpec: ADR-004)

- **UT-045** (happy): the embedded filesystem contains exactly the nine intended `skills/<name>/SKILL.md` resources, the manifest declares `resources.skills: ["skills"]`, and `LoadManifest` normalizes that resource. `cy-capture-decisions` is absent from this bundle. This file-content assertion belongs in the existing embed suite because the embedded bytes are the shipped product contract, not an implementation snapshot.
- **UT-046** (happy): each bundled `SKILL.md` parses through the extension skill source, proving compatible frontmatter rather than only embedded-file presence.
- **UT-047** (state): bundled bodies contain no retired CLI invocations (`compozy tasks run`, `compozy reviews fix`, `compozy setup`), no bundled skill named `compozy`, and no `cy-capture-decisions` resource or invocation. This narrow content assertion is permitted because the embedded bundle is a shipped product contract and `embed_test.go` already owns it.

### Config migrator (`internal/cli/migrate_config.go`; TechSpec: ADR-006)

- **UT-048** (happy): a v0.2.15 `[defaults]` fixture expands supported `ide`, `model`, and `reasoning_effort` into the four delivery/watch worker/judge runtime-default paths, maps `timeout` to `session.limits.timeout`, and maps explicit `auto_commit` to both declared loop inputs. The provider table covers all eight supported legacy IDEs including `cursor-agent -> cursor`; `droid`, `devin`, and `ultra` emit unsupported-value drops without lossy aliases. `[defaults.by_complexity.<low|medium|high|critical>]` becomes delivery runtime rules in canonical order. The migrator never invents a nonexistent `defaults.pr` value.
- **UT-049** (happy): v0.2.15 `[[tasks.run.task_runtime_rules]]` accepts persisted type-only rules and preserves their order and field overrides in `[loops.defaults.delivery].runtime_rules`. Task-id rules are not fabricated from a legacy schema that never persisted them.
- **UT-050** (happy/state): review-key handling is drop-dominant (ADR-008): `defaults.auto_commit` still maps to both declared loop inputs, while **every** `watch_reviews.*` and `fetch_reviews.*` key — `provider`, `nitpicks`, `auto_push`, `push_remote`, `poll_interval`, `quiet_period`, `max_rounds`, `review_timeout`, `until_clean`, `push_branch` — is a report-only drop with a guide reference, never translated into a loop input or invented default; unsupported `fix_reviews` fields are reported according to the exhaustive drop matrix.
- **UT-051** (error): unparseable legacy TOML fails structurally with file and line, exits non-zero, and writes no output.
- **UT-052** (boundary): a file containing only deferred keys emits a valid empty config and a drop-only report.
- **UT-053** (state): a fixture populating every leaf in the authoritative v0.2.15 schema proves each source leaf appears exactly once in `-o json`, with ordered `targets` for one-to-many translations. The fixture covers all remaining scalar/stall defaults; `tasks.types`; every task-run selection, multiple-run, parallel, and conflict-resolver leaf; every review leaf; every embedded `exec` runtime/stall leaf plus `verbose|tui|persist`; and all `runs`, `recovery`, and `sound` leaves. Every non-translated leaf has `{key, action: dropped, reason, guide_ref}`; no category-only receipt can hide a nested omission.
- **UT-054** (state): the original file is preserved at a timestamped backup path before in-place output replacement.
- **UT-055** (idempotency): a second invocation sees the backup marker and refuses with a precise error, preventing double translation.

### Deep link and rebrand constants

- **UT-056** (happy/boundary): a successful non-dry `loop run` emits its final human-readable line as `http://<configured-host>:<configured-port>/loop-runs/<run_id>` and emits the same optional `web_url` through JSON and TOON. Dry-run has no run ID and omits the human link and `web_url`; the path comes from the shared real-route contract, not a guessed SPA fallback.
- **UT-057** (state): home resolution uses `DirName=".compozy"`, honors `COMPOZY_HOME`, and uses database filename `compozy.db`; existing home/config suites are updated rather than duplicated.
- **UT-058** (state): the existing reserved-prefix guard rejects extension collisions under `compozy__`.
- **UT-059** (state): `ProtocolV0` is exactly `compozy-network/v0`, and all existing envelope extension keys use the `compozy.*` namespace. Existing network transport/envelope suites co-ship the hard cut; no nonexistent NATS subject-prefix constant is introduced.

### Per-loop input defaults (`[loops.inputs.<name>]`; TechSpec: ADR-007, Invariant 14)

- **UT-063** (happy): an absent declared run input resolves from workspace `[loops.inputs.review-and-fix].auto_commit = true`; when both global and workspace specify it, workspace wins per key; untouched inputs keep their definition default.
- **UT-064** (error): an unknown configured input key or explicit per-run input key produces typed `input_default` `{loop, key, reason: unknown_input}` when effective dry-run/submission resolves the named loop; static definition-only `loop validate` does not pretend to evaluate unavailable workspace values, and no input source silently drops unknown keys.
- **UT-065** (error): a string for a Boolean input produces typed `input_default` at effective submission and rejects the run before spawn.
- **UT-066** (boundary): a required input such as `review-and-fix`'s `task_name` can be satisfied by configuration; an explicit run input always wins over every config layer.

## Integration Tests

### Runtime selection engine and binder (`acpmock`)

- **IT-001**: a complete loop run over three task fixtures—`type: frontend`, `type: docs` with frontmatter runtime, and bare—plus per-run `--runtime` rules gives each `acpmock` session the expected `CreateOpts{Provider, Model, ReasoningEffort}` and records per-task provenance.
- **IT-002**: two tasks resolving different providers bind through isolated/ephemeral paths; forcing a run-owned binding with a divergent triple returns `bindingMismatch` (Invariant 4).
- **IT-003**: `POST /api/workspaces/{workspace_id}/loops/{name}/validate` over HTTP and UDS validates static loop-definition errors with a 422 body containing `runtime_validation` items `{field, value, reason}`. An effective unknown provider or authoritative-catalog model from workspace/run values is instead rejected by dry-run/submission before spawn, while a non-authoritative provider model is not falsely rejected; the two validation phases are not conflated.
- **IT-004**: a provider override with `bound_secret` policy but no slot fails binding and spawns no ACP process, as observed by the `acpmock` ledger.
- **IT-005**: a gate criterion runtime model different from `runtime_defaults.judge` starts the judge session with the criterion model, proving real evaluator wiring.
- **IT-006**: `compozy loop run --workspace <ref> --name <name> --runtime … --dry-run -o json` resolves effective layers and echoes them; submitting the equivalent request through HTTP and UDS yields the identical effective runtime and provenance. This is the owner for dynamic, not static, validation.
- **IT-007**: after IT-001, the append-only global-stream migration and canonical fresh/reopen/equivalence suites prove `resolved_runtime` persists through a real store close/reopen without rewriting existing migration history. The value is consistent in `GET /api/workspaces/{workspace_id}/loop-runs/{run_id}` over HTTP/UDS, `compozy loop status --workspace <id> --run-id <id> -o json`, `compozy__loop_status`, and run SSE. A second workspace cannot list, read, receive, or invoke a run from the first workspace.
- **IT-008**: the real loader merges `runtime_defaults` and `runtime_rules` across delivery/watch defaults and stored config in documented precedence order, including complexity; `model_defaults` and legacy scalar `params.model` fail strict load/validation with migration-guide remediation.

### `review-and-fix` pipeline (`acpmock` reviewer/fixer plus trusted temporary workspaces)

- **IT-009**: the reviewer agent session (via `acpmock`) returns two structured `ReviewIssue`s through its `output_schema`, then `write_review_artifacts` writes byte-golden files under the caller workspace's `.compozy/tasks/<task>/reviews-001/` and fans out issue-file paths. A second trusted workspace with the same task name receives its own artifacts; symlink/traversal attempts never escape either workspace.
- **IT-010**: a stub fixer marks one file `valid` and one `invalid`; finalize changes both to `resolved`, returns `{resolved:2, invalid:1, pending:0}` where invalid is retained as the pre-finalize audit count, and reconstructs round metadata solely from frontmatter without `_meta.md`.
- **IT-011**: a reviewer session emitting schema-invalid issues (missing `title`, `body`, or `severity`) through `acpmock` fails the review node with a structured schema error; no round directory is created and no partial artifacts are written.
- **IT-012**: the full `review-and-fix` graph via `acpmock`: generation 1's review emits issues → artifacts are written → the fixer batch triages/remediates → finalize resolves the round → generation 2's review returns zero issues → the loop ends `done` with no watch, thread-resolution, or push node executed (none exist in the graph), covering two-workspace isolation.

### Extension, migrator, and release identity

- **IT-013**: dev-cycle boot enrollment publishes `extension/dev-cycle/skills` as the intentional global resource. Two workspace-scoped sessions independently enumerate/view the same immutable nine bundled skills without workspace-local resource leakage; `compozy skill view cy-execute-task` returns the bundled body, while `cy-capture-decisions` is unavailable. The bundled-source classification bypasses `VerifyContent`, whereas an otherwise identical non-bundled skill is scanned; changing embedded skill bytes changes the manifest checksum/version enrollment identity and causes the updated bundled resource to re-enroll.
- **IT-014**: run `compozy migrate config` against the global v0.2.15 fixture, then run `compozy migrate config --workspace <root>` against the workspace fixture. Each independently reported invocation touches only its selected target; together they cover type-only rules, complexity defaults, and the complete review/drop matrix, emit configurations the real v0.3.0 loader accepts, and report every translation/drop exactly once per source. Each rerun refuses. Before migration, daemon startup and local `compozy status -o json`/`doctor -o json` return the same `legacy.state_detected` remediation while HTTP/UDS are unavailable because the daemon cannot boot. Legacy-owned global/workspace `agents/` and `extensions/` collision fixtures are listed and never loaded or enrolled. After migration, the running daemon retains orphan paths and exposes the same structured orphan warning through CLI, HTTP, and UDS status/doctor payloads.
- **IT-015**: a successful non-dry `compozy loop run` against a daemon bound to a configured random HTTP port emits a deep link matching the shared `/loop-runs/{run_id}` route contract; the real HTTP route is reachable and the URL uses the effective listener host/port. Dry-run omits the link because no run exists. A bare SPA-index 200 is not accepted as route proof; rendered-route truth belongs to E2E-004.
- **IT-016**: post-rename smoke boots the daemon in a temporary `COMPOZY_HOME`, creates `compozy.db`, listens on the renamed UDS socket, and makes `compozy status -o json` report the new paths; it also resolves the renamed native loop tool IDs through the real descriptor surface, not a grep assertion.
- **IT-017**: `goreleaser check` passes against the switched configuration; the release workflow and vendored release skill pin `github.com/compozy/releasepr@v0.0.24`; and a release-PR dry-run invokes `pr-release plan` with authoritative `ref=<candidate>`, `version=0.3.0-beta.1`, and `channel=beta`. The planner resolves the ref to checked-out `HEAD`, rejects a leading-`v` version, derives `release_tag=v0.3.0-beta.1` exactly once, and fails when that tag already exists locally or on `origin`. The dry run proves the workflow consumes, without re-derivation, `release_ref`, `release_commit`, `release_version`, `release_tag`, `release_channel`, `github_prerelease=true`, `github_make_latest=false`, `npm_tag=beta`, and `homebrew_skip_upload=true`; passes the unprefixed version to GoReleaser/npm; exposes no AUR publisher; and passes installer checks for `RELEASE_REPO=compozy/compozy`, the Cosign identity regexp, and `COMPOZY_*` names. Companion table cases prove `stable` and `legacy` emit non-prerelease/latest, npm `latest`, and Homebrew-publish policy while the checked-out legacy branch retains ownership of v0.2 artifact identity. The planner remains read-only: the Compozy workflow owns annotated tag creation and publication. It does not claim an external release was created; Task 10's post-publish runbook owns that live assertion. The release-PR dry-run remains separate from local verify/integration/E2E lanes.
- **IT-018**: effective input defaults follow the one resolver shared by dry-run and every closed `StartKind` (`manual|cli|http|uds|trigger|schedule|webhook|network|extension|native_tool`). Workspace `[loops.inputs.review-and-fix] auto_commit = true` appears in `compozy loop run --workspace <ref> --name review-and-fix --dry-run -o json` with workspace origin, while `task_name` has run origin. Explicit `--input auto_commit=false` wins; an unknown configured or per-run key fails effective dry-run/submission with typed `input_default` over HTTP and UDS.

## End-to-End Tests

### Legacy delivery journey (P0-2/3/6; migration-guide command map)

- **E2E-001** (`make test-e2e-runtime`): seed `.compozy/tasks/demo/` with a `compozy.tasks/v2` graph and three tasks with mixed `type`/frontmatter runtime → `compozy loop run --workspace <workspace_id> --name software-delivery --input slug=demo --runtime type=frontend:claude/opus` completes against `acpmock`; per-task runtimes are visible through `compozy loop status --workspace <workspace_id> --run-id <run_id> -o json`, and the real deep link is printed.

### Review remediation journey (P0-1)

- **E2E-002** (`make test-e2e-runtime`): a seeded workspace task with reviewable work → `compozy loop run --workspace <workspace_id> --name review-and-fix --input task_name=<slug>` runs the reviewer agent (via `acpmock`) which emits fixture issues, writes `reviews-001/`, the stub fixer remediates, statuses finalize to `resolved`, the next review round returns zero issues, the run ends `done`, and artifacts match the v0.3 byte contract.

### In-place upgrade journey (P0-4/5)

- **E2E-003** (`make test-e2e-runtime`): fixture machine state with a v0.2.15 legacy `~/.compozy`, legacy workspace config plus colliding legacy `agents/`/`extensions/`, and no `.agh` fallback → the preflight refuses to load legacy-owned resources → `compozy migrate config` for global plus a separate `compozy migrate config --workspace <root>` for the workspace → first daemon boot reports but never deletes orphan legacy paths → `software-delivery` runs with translated runtime rules → `review-and-fix` dry-run echoes the translated workspace `auto_commit` default with its origin.

### Web observation journey (P0-6; SD-007 truthful UI)

- **E2E-004** (`make test-e2e-web`): open the emitted `/loop-runs/<run_id>` deep link for a seeded run → the real run page renders each task's resolved provider/model/reasoning and provenance exactly as the status API reports; no runtime-edit control is rendered because runtime is display-only.
