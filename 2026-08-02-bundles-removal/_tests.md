# Test Specification: Bundles Removal — Extension as the Single Kit Unit

Canonical test contract for the bundles-removal program. Companion to `_techspec.md`. Derived from `_brief.md` requirements (keys R-HC-1..7, R-P0-1..5, R-P1-1..5, R-AM-1..4 — no `_user_stories.md` exists for this program; end-to-end journeys are derived from the brief's Requirements and Verification sections, and that gap is accepted) and from `_techspec.md` components/ADRs.

## Strategy

- Frameworks: Go `testing` with `-race` + `CGO_ENABLED=1` (`t.Run("Should …")` subtests, `t.Parallel` default, table-driven where shapes repeat); `+integration` build tag for wiring suites; Go E2E runtime harness (`acpmock` + fixture extensions); Vitest for web units; Playwright for browser E2E; site Bun tests for docs truth.
- Fakes only at I/O boundaries: vault crypto uses the real service over a temp store; subprocess transport faked at the extension protocol boundary; scheduler time faked via its existing clock seam; no mocking of `resources` kernel or automation manager internals.
- Assertions: status code **and** body for every transport case; structured CLI output compared as decoded JSON (goldens for human mode); event assertions via the existing test notifier at the emitting call site; secret-leak assertions grep serialized payloads/events/logs for planted sentinel values.
- Removal correctness is owned primarily by compilation (deleted types/routes cannot be referenced) plus the updated goldens (OpenAPI, native-tool catalog, site tests) — only gaps compilation cannot catch get explicit IDs below.
- Modified-not-new gates (no IDs, tracked in Suite Placement): existing suites updated for the cut — transport route tables, marketplace kind tests, toolmeta/catalog goldens, site docs tests, `internal/resources` fixture-kind renames.

## Coverage Matrix

| Source | Behavior | Unit | Integration | E2E |
| --- | --- | --- | --- | --- |
| R-HC-1 | No public Bundle surface remains (routes/CLI/tools/kind/web) | UT-055, UT-057, UT-058, UT-062 | IT-011, IT-012 | E2E-004, E2E-005 |
| R-HC-2 | No authoring path requires bundles for a kit | UT-043–UT-046 | IT-003–IT-005 | E2E-001 |
| R-HC-3 | Docs/skill/OpenAPI/CLI refs stop teaching Bundle | — (site suites + codegen-check own it) | — | — |
| R-HC-4 | Homonyms keep working | — | IT-016 | E2E-006 |
| R-HC-5 | Disable/remove no longer gated on activations | UT-057 | IT-004 | E2E-001 |
| R-HC-6 | QA scenarios rewritten/removed | — (docs/qa tree change, verified by qa-report task) | — | — |
| R-HC-7 | Removal inventory fully unwired; orphan rows cleaned | — | IT-002 | — |
| R-P0-1 | Agents + SOUL/HEARTBEAT live on enable | UT-001–UT-008, UT-019–UT-021 | IT-003 | E2E-001, E2E-002 |
| R-P0-2 | Package jobs/triggers; enable-as-consent; no start on install | UT-009–UT-014, UT-047, UT-061, UT-068, UT-070 | IT-004 | E2E-001 |
| R-P0-3 | Package window layouts on enable | UT-015–UT-017 | IT-005 | E2E-001 |
| R-P0-4 | Enable/disable is the kit switch (owner-attributed sweep) | UT-018, UT-059 | IT-003–IT-006, IT-017 | E2E-001, E2E-002 |
| R-P0-5 | Missing-env reporting + binding path | UT-022–UT-031, UT-048, UT-054, UT-065 | IT-007, IT-008, IT-015, IT-018, IT-020 | E2E-001 |
| R-P1-1 | Resource inventory (shipped vs live) | UT-038, UT-049, UT-069 | IT-011, IT-012 | E2E-001, E2E-003, E2E-004 |
| R-P1-2 | Enable preview without mutation | UT-039–UT-042 | IT-011, IT-012 | E2E-003 |
| R-P1-3 | Network confirm on extension lifecycle | UT-032–UT-037, UT-050, UT-052, UT-066, UT-067 | IT-009, IT-010, IT-014, IT-018 | E2E-001 |
| Lifecycle coordinator (B-002) | Serialized mutation + reverse rollback | — | IT-019 | — |
| R-P1-4 | Update reconciles kit resources (no stale projections) | — | IT-006, IT-010 | E2E-001 |
| R-P1-5 | Provides truthfulness (closed-set; no decorative teaching) | — (owned by existing ext-improvs closed-set suites; docs strips in F) | — | — |
| R-AM-1 | CLI/HTTP/UDS/native parity for kit management | UT-049, UT-051 | IT-011, IT-012 | E2E-003 |
| R-AM-2 | Site docs co-ship the cut | — (site suites) | — | — |
| R-AM-3 | Official skill updated | — (rune-budget gate + docs review) | — | — |
| R-AM-4 | Agent instruction checklists updated | — (editorial; verified in phase F review) | — | — |
| Migration 00029 | Additive schema (bindings, confirm columns) | — | IT-001 | — |
| Events matrix | New events with required keys, no leaks | UT-060, UT-061 | — | — |
| Deterministic errors | 409/400 contracts | UT-052–UT-054, UT-050 | IT-009, IT-013 | — |
| Agent-name conflicts | Preview shows; enable fails closed | UT-040 | IT-013 | — |
| Web inventory/confirm UI | Truthful rendering of new payloads | UT-063, UT-064 | — | E2E-004 |

## Unit Tests

### Static agent loader (TechSpec: Core Interfaces / ADR-007)

- **UT-001** (happy): `LoadAgentResources` on `agents/writer/{AGENT.md,SOUL.md,HEARTBEAT.md}` — returns one `StaticAgent` with `Agent.Name == "writer"`, both sidecars non-nil with parsed bodies and `SourcePath` pointing at the package files.
- **UT-002** (happy): agent dir with only `AGENT.md` — both sidecars nil, no error.
- **UT-003** (error): loose `agents/stray.md` directly under the entry — deterministic error naming the expected `<entry>/<agent>/AGENT.md` layout; nothing loaded.
- **UT-004** (error): agent dir without `AGENT.md` — error naming the dir and the missing file.
- **UT-005** (error): `SOUL.md` with invalid frontmatter — register-phase error naming the file and the parse reason.
- **UT-006** (error): `HEARTBEAT.md` with a non-allowlisted frontmatter key — error naming file + key.
- **UT-007** (boundary): agent dir symlinked outside the extension root — containment rejection (same guard behavior the bundle loader had).
- **UT-008** (happy): `mcp.json` + `capabilities.toml` beside `AGENT.md` — merged into the loaded `AgentDef` (LoadAgentDefFile semantics preserved).

### Package automation loader (TechSpec: Core Interfaces / ADR-002)

- **UT-068** (error): job/trigger whose `agent` names an agent NOT shipped by the same extension (an authored workspace agent, another extension's agent) — deterministic load/validate error naming the shipped agent set (R1 N-003).
- **UT-009** (happy): TOML with `[[jobs]]` + `[[triggers]]` — parsed with `Enabled` defaulting true, schedule/retry/fire_limit populated.
- **UT-010** (error): trigger with `event = "webhook"` — rejected with the carried-over unsupported-webhook error.
- **UT-011** (error): two jobs named `daily` across the extension's automation files — duplicate-name error naming both files.
- **UT-012** (error): job with invalid cron `expr` — the `automation.Job.Validate` failure surfaces with file path context.
- **UT-013** (happy): materialization mapping produces namespaced names `<ext>/<name>`, `Source: package`, `Owner{extension}`, global scope.
- **UT-014** (state): job with `enabled = false` — materialized record `Enabled == false`; excluded from `AutomationStarting` in preview.

### Layout loader (TechSpec: Core Interfaces)

- **UT-015** (happy): valid layout `.json` — decoded via the windowmanager codec at global scope; `Document.WorkspaceID` empty.
- **UT-016** (error): a `.toml` layout path or a directory-as-file — rejected naming the requirement (`.json` regular file).
- **UT-017** (error): layout JSON failing codec validation (missing `id`) — error carries the codec reason.

### Owner attribution + catalog semantics (TechSpec: Data Models / ADR-004)

- **UT-018** (happy, table-driven over all kit kinds): publisher desired-resource construction stamps `Owner{Kind:"extension", ID:<name>}` + the actor pair on agents, souls, heartbeats, skills, loops, tools, mcp_servers, hook bindings, jobs, triggers, layouts.
- **UT-019** (state): `ResolveAgentArtifacts` sets `PackageOwned == true` for `Owner.Kind == "extension"` records and false for unowned records.
- **UT-020** (happy): soul/heartbeat catalog matching succeeds when sidecar and agent share `AgentResourceID` + scope + extension owner; `ResolveHeartbeatPolicy` returns the parsed policy.
- **UT-021** (error): sidecar with a different owner ID does not match the agent (no cross-extension leakage).
- **UT-059** (state): managed-record equality (`sameManagedRawRecord` + per-kind peers) treats a row identical in scope+spec but lacking the extension owner (or carrying a different source) as NOT current — the sync rewrites it (R1 B-004; closes the former ID gap).

### Env bindings (TechSpec: Core Interfaces / ADR-003)

- **UT-022** (happy/boundary): `ExtensionSecretRef("my-ext","","API_KEY")` == `vault:extensions/global/my-ext/env/API_KEY` and `ExtensionSecretRef("my-ext","ws-1","API_KEY")` == `vault:extensions/ws/ws-1/my-ext/env/API_KEY`; unsafe segments (`My Ext!`) hex-encode collision-safely (mirror of the MCP namer tests).
- **UT-023** (happy): binding store put/list/delete round-trip preserving kind + timestamps; PK upsert on same `(extension, workspace_id, env)`; `''` and `ws-1` rows for the same extension+env coexist independently.
- **UT-024** (error): `SetSecrets` with an env name not in `requires_env` — 400-class error listing the declared names; nothing written.
- **UT-025** (error): `Bind` with `vault:mcp/...` ref — namespace rejection.
- **UT-026** (error): `Bind` to a ref whose vault metadata is absent — dangling-ref rejection; no binding row created.
- **UT-027** (idempotency/ordering): `SetSecrets` with three env names — mutation order is the sorted normalized names; an injected failure on the second write rolls back the first to its prior value/kind in reverse order with binding rows unchanged; the same env name carrying both value and ref forms → 400 before any write (R1 N-006).
- **UT-028** (state): re-`set` after a ref change GCs the superseded `extension_env`-kind ref only when no other binding references it; foreign-kind refs never deleted.
- **UT-029** (ordering): `resolveEnvMap` merges allowlist → manifest `env` → manifest `secret_env` → the launching instance's own bindings; a binding for a name also in authored `secret_env` wins; a global instance ignores `ws-1` rows and a dev instance ignores `''` rows (no cross-scope fallback); output ordering deterministic.
- **UT-030** (error): binding whose ref no longer resolves at spawn — launch aborts with an error naming the env key and ref; redaction cleanups unwound.
- **UT-031** (state): `MissingEnv` = declared − (process env ∪ non-stale bound keys); bound-but-unset-in-process names are NOT missing.
- **UT-065** (state): after an update drops `API_KEY` from `requires_env`, its binding row is reported `stale: true` by `secrets list`, excluded from spawn injection and from `MissingEnv` math, and removed only by explicit `secrets unset` (R1 B-003).

### Network confirm (TechSpec: Core Interfaces / ADR-005)

- **UT-032** (happy): `ConfirmNetworkRequirement(InstanceKey{name,""}, expectedDigest, actor, now)` with the matching current digest stores `{digest, actor, RFC3339Nano}` on the `extensions` row; `NetworkConfirmation` reports valid while manifest digest equals stored digest.
- **UT-033** (error): enable gate with non-empty unconfirmed digest — `ErrExtensionNetworkConfirmationRequired` wrapping the **current** digest hex.
- **UT-034** (state): manifest digest change → stored confirmation reported invalid; gate re-arms.
- **UT-035** (state): disable → enable with unchanged digest — no re-confirmation required.
- **UT-036** (error): `ConfirmNetworkRequirement` with an `expectedDigest` that doesn't match the re-read current manifest digest — rejected carrying the current digest; nothing recorded (a stale digest is never ratified, R1 B-006).
- **UT-037** (boundary): manifest without `network_participation` — digest empty, no gate, columns untouched.
- **UT-066** (error): enable/update request with a stale `confirm_network_digest` (manifest changed after the caller previewed) — 409 `extension_network_confirmation_required` whose payload carries the current digest as the remediation.
- **UT-067** (state): confirmation via an operator transport records `confirmed_by == "operator"`; the same confirmation via `compozy__extensions_enable` records `confirmed_by == "agent:<session-scoped id>"` from the canonical dispatcher — never the operator literal (R1 B-006).

### Inventory + preview (TechSpec: Core Interfaces / ADR-006)

- **UT-038** (happy): inventory for an enabled kit — shipped∪live items per kind with `Live == true`; for a disabled kit — same items with `Live == false` and the gate explanation.
- **UT-039** (happy): preview composes `WouldPublish`, `MissingEnv` (unbound only), `AutomationStarting` (effective-enabled only), digest + `NetworkConfirmationRequired`.
- **UT-040** (error): shipped agent name colliding with a visible authored agent — preview lists it in `AgentConflicts` AND enable fails 409 with the same set (equivalence assertion).
- **UT-041** (state): preview performs zero mutations — no records written, no vault writes, no subprocess start, no confirmation recorded (spy-asserted).
- **UT-042** (happy): preview of an enabled extension after a staged kit change reports the reload delta (added/removed items).
- **UT-069** (state): KitItem identity is `(Kind, Name)` — a content-changed shipped resource keeps one inventory row (same source-key ID, rendered as changed in preview), while a renamed resource splits into remove+add (R1 N-002).
- **UT-070** (state): `ExtensionEnableResult.AutomationStarted` contains exactly the effectively-enabled definitions made runnable by the committed operation (authored enabled × overlays), name-sorted; a job disabled by overlay is absent; the list never appears in `ExtensionPayload`/status/list responses (R1 B-005).

### Describe/build resources (TechSpec: Core Interfaces / ADR-008)

- **UT-043** (happy): `DescribePayload.resources` → generated `extension.toml` `[resources]` entries for skills/loops/agents/automation/layouts.
- **UT-044** (happy/idempotency): build with declared resource dirs copies them into the generation; identical source → byte-identical manifest (determinism invariant extended).
- **UT-045** (error): declared automation TOML with an invalid job — `compozy extension build`/`validate` fails with file position; generation not produced.
- **UT-046** (happy): handwritten resource-only manifest declaring `automation`/`layouts` validates; declaring `bundles` fails with unknown-field error (hard cut, no alias).

### CLI output (TechSpec: Agent Manageability)

- **UT-047** (happy): `extension enable` human + JSON output renders `ExtensionEnableResult` — started automation enumerated (namespaced names) + next-step hints; empty kit → no automation section; JSON output is the wire `ExtensionEnableResult` verbatim.
- **UT-048** (happy): `extension secrets list` shows declared env, bound keys, and never values/refs' secret content (golden).
- **UT-049** (happy): `extension inventory`/`preview` human table + `-o json` structural goldens.
- **UT-050** (error): confirm-required failure renders via `renderHumanExecutionError` with the digest and the `--confirm-network-requirement` retry suggestion.
- **UT-051** (happy): `-o jsonl` and `-o toon` parity for `inventory`, `preview`, `secrets list`.

### Contract + error mapping (TechSpec: API Endpoints)

- **UT-052** (error): `ErrExtensionNetworkConfirmationRequired` → 409 `extension_network_confirmation_required` with digest in the body.
- **UT-053** (error): agent conflict → 409 `extension_agent_conflict` naming the agents.
- **UT-054** (error): undeclared binding name and dangling ref → 400 with the documented codes; vault sentinel mappings unchanged.
- **UT-055** (state): `ExtensionPayload` serialization — `bound_env_keys`, `network_requirement_digest`, `network_confirmation_required` present; no `bundles` key (compile + JSON round-trip).
- **UT-056** (state): skill payload/spec round-trip carries no `installed_from_bundle` (updated canonical skill suites).
- **UT-057** (state): surfaces registry lookup/`PublishableKinds` contains no `bundle`/`bundle.activation`; extension disable/remove path has no activation gate.
- **UT-058** (state): native-tool catalog golden — `compozy__extensions_inventory`/`_preview` rows present (read-only risk class), zero `compozy__bundles_*` rows, no `bundles.read`/`bundles.write` capabilities, enable tool input schema carries `confirm_network_digest`.

### Events (TechSpec: Monitoring)

- **UT-060** (state): `extension.secrets.updated` + `extension.network.confirmed` emit with exactly their required keys at the emitting call sites; planted sentinel secret values do not appear in any event payload.
- **UT-061** (state): `extension.enabled` carries `automation_started_count` matching the enumerated enable output.

### Web (Vitest)

- **UT-062** (state): marketplace kind system exposes exactly `extension|skill|mcp` — kind config, route kinds, discriminated unions, and MSW handlers have no bundle arm (updated shared suites).
- **UT-063** (happy): extension detail inventory panel renders kit items (kind, name, live badge) from a mocked `GET /api/extensions/{name}/inventory`.
- **UT-064** (happy): enable flow shows the network-confirm affordance when the payload sets `network_confirmation_required`, and passes `confirm_network_digest` on the enable mutation.

## Integration Tests

### Migrations (canonical globaldb suites)

- **IT-001**: migration `00029` — fresh apply, reopen with preserved data, ahead-version refusal, integrity, declarative-equivalence; `extension_env_bindings` PK enforced; confirm columns default correctly.
- **IT-002**: migration `00030` on a DB seeded with `bundle`/`bundle.activation` records AND activation-owned agent/soul/job records AND homonym rows (support-bundle-unrelated kinds, `SourceBundled` skills) — bundle-kind and activation-owned rows deleted, everything else byte-preserved; reopen clean.

### Kit publish/unpublish round trips (daemon wiring)

- **IT-003**: enable a fixture extension shipping `agents/writer/{AGENT.md,SOUL.md,HEARTBEAT.md}` — `agent` + `agent.soul` + `agent.heartbeat` records exist with the extension owner; `ResolveAgentArtifacts` returns both bodies; heartbeat policy resolves; authored-context API reports the agent read-only (PackageOwned); disable deletes all three.
- **IT-004**: enable a fixture shipping jobs (one enabled, one `enabled = false`) + a trigger — enabled job registered with the scheduler and fires under a faked clock; disabled job present but unregistered; automation API rejects mutation (`ErrDefinitionReadOnly`) but accepts an `enabled_override` overlay; disable deletes records, unregisters, and GCs the overlay row; extension remove no longer errors on any activation state.
- **IT-005**: enable a fixture shipping a layout — `window_layout` record listed by the layout-profiles route; `ArrangeLayoutCommand{ResourceID}` applies it; disable removes it from the list.
- **IT-006**: `extension update` swapping to a kit version that drops one job and adds one skill — post-reload records converge (stale job deleted, its overlay GC'd, new skill live); no torn state mid-reload observable through the catalog.

### Secrets binding (real vault, temp store)

- **IT-007**: `PUT /api/extensions/{name}/secrets` with one value-form + one ref-form input against the **global instance** → rows keyed `(name, '', env)` + vault writes at `vault:extensions/global/...`; subprocess spawn env contains the resolved values under the declared names; sentinel value absent from status/logs/SSE payloads. Dev variant: the same call through the dev flow binds `(name, <workspace>, env)` at `vault:extensions/ws/...` and only the dev instance's spawn sees it.
- **IT-008**: injected vault failure mid-batch over the transport — response is the mapped error, prior secret values restored, `GET .../secrets` shows the pre-call state.
- **IT-015**: doctor extension probe on a kit with one unbound required env — emits the `missing_env` diagnostic whose suggested command is `compozy extension secrets set <name> --env <KEY>`.

### Confirm flows

- **IT-009**: HTTP + UDS enable without confirm on a Live-declaring fixture → 409 with the current digest; with `confirm_network_digest` equal to it → 200 returning `ExtensionEnableResult`, columns persisted with the caller's real actor, `extension.network.confirmed` emitted; with a stale digest → 409 carrying the current one; disable/enable cycle needs no re-confirm.
- **IT-010**: update staged to a manifest with a changed digest on **marketplace-managed sources** (curated + github/git against mocked registries; local-path installs have no update lifecycle — R1 B-007) — refusal happens **before** swap (old version still installed and running, registry/files/confirmation byte-identical); the single verb retried with the matching `confirm_network_digest` applies and re-records; `--all` refuses the digest-changing item per-item with its digest in the batch partial progress and confirms nothing.
- **IT-014**: dev-lane reload where the new generation's manifest changes the digest — reload refuses with the same 409 contract until confirmed; the confirmation persists on the `extension_dev_links` row and works for a dev-only extension with **no** `extensions` row (R1 B-001).

### Transport + tools parity

- **IT-011**: route-table parity suites updated — `inventory`/`preview`/`secrets` registered on HTTP and UDS with identical shapes; `/api/bundles/*` absent from both routers and from the OpenAPI operation registry (spec test asserts the tag and 8 operations are gone).
- **IT-012**: native tools — `compozy__extensions_inventory`/`_preview` outputs deep-equal the route payloads; enable tool passes the confirm field; bundle tool IDs absent from availability, bindings, and toolmeta.
- **IT-013**: agent-conflict enable — fixture agent name colliding with a pre-seeded authored agent → enable 409 naming it; preview beforehand listed the same conflict; reserved builtin names (coordinator etc.) rejected identically.

### Owner convergence, isolation, lifecycle serialization, binding retirement

- **IT-017**: seeded pre-cut records — for every kit kind, a stored row with today's shape (daemon actor, no owner, identical spec) converges on the first reconcile: rewritten with `Owner{extension}`, sidecars match, inventory reports it live, and a subsequent disable deletes it (R1 B-004).
- **IT-018**: cross-workspace non-leakage — bindings and confirmations created for the global instance and for workspace `ws-1` are invisible to each other and to `ws-2`: `secrets list`, spawn injection, `NetworkConfirmation`, and the reload gates each resolve only their own instance's state (R1 B-001; L-033).
- **IT-019**: lifecycle serialization — concurrent enable/update/disable on one extension interleave as whole operations (per-name coordinator); injected failures at each stage (staged mutation, confirmation write, reload, reconcile) roll back to byte-identical persisted and running state, including restoring the prior confirmation tuple after a failed update that had recorded a new one (R1 B-002).
- **IT-020**: binding retirement — an update that drops a declared env leaves its row stale (listed, never injected); `extension remove` (global) and `dev unlink` (workspace) delete the instance's rows and GC only owned `extension_env` refs, preserving a foreign ref another subsystem points at; reinstall after remove starts with zero bindings (R1 B-003).

### Homonym smoke

- **IT-016**: post-cut: `POST /api/support/bundles` + status/download round-trip unchanged; `compozy skill list --source bundled` precedence unchanged; task ContextBundle payload unchanged (existing canonical suites keep passing — cited here as the gate, not duplicated).

## End-to-End Tests

### Kit lifecycle journey (brief Verification; R-P0, R-P1)

- **E2E-001** (runtime harness): install fixture kit extension from a mocked marketplace source → `extension secrets set` (stdin value) → `extension enable --confirm-network-requirement <digest>` (output enumerates 1 started job) → `extension inventory` shows every shipped item live → invoke a kit tool → stage update with changed digest → `extension update` refused pre-swap with the new digest → retry with `--confirm-network-requirement <new-digest>` succeeds → `extension disable` → inventory shows nothing live, scheduler empty, layout gone.
- **E2E-002** (runtime harness): dev-cycle regression — enable exposes its skills/loops/agents (dir-per-agent) /tools in catalogs exactly as before the program; disable removes them.
- **E2E-003** (runtime harness, agent-driven): with required secret bindings pre-seeded by the operator setup (secret writes intentionally have no native tool), run the remaining journey through native tools only (`extensions_install/…/enable` with confirm field, `extensions_inventory`, `extensions_preview`), asserting structured outputs and that no secret value appears in any tool transcript.

### Web journeys

- **E2E-004** (Playwright): marketplace renders 3 kinds with shared shells working; extension detail shows the kit inventory panel and the bound-env presence; enabling a Live-declaring extension walks the confirm affordance.
- **E2E-005** (Playwright): the former bundle acquisition journey in `marketplace.spec.ts` is replaced by the extension-kit journey (install → enable → inventory → disable) with the bundle selectors/fixtures deleted.

### Homonym journey

- **E2E-006** (runtime harness): `compozy support bundle` create → status → download works end-to-end after the cut.

## Suite Placement (per `eng-consolidate-test-suites`)

| Case group | Owning canonical suite |
| --- | --- |
| UT-001..008 | `internal/extension/agent_resources_test.go` (new, co-located with the new loader) |
| UT-009..014, UT-068 | `internal/extension/automation_resources_test.go` (new) |
| UT-015..017 | `internal/extension/layout_resources_test.go` (new) |
| UT-018 (shared owner constructor, tested once) + UT-059 (owner-aware equality) | the substrate's own suites: `internal/resources` typed/kernel suites for the Draft owner + the publisher-helper package's suite for `extensionOwner`/equality; provider suites assert only their per-kind domain mapping (R1 N-004) |
| UT-019..021 | `internal/daemon/agent_skill_resources_integration_test.go` + catalog suites (extend existing) |
| UT-022 | `internal/vault/extension_refs_test.go` (new, mirrors `mcp_refs_test.go`) |
| UT-023 | `internal/store/globaldb` canonical suite (sqlc binding queries — persistence invariants live with the store, R1 N-004) |
| UT-024..028 | `internal/daemon/extension_secrets_test.go` (transactional service) (new) |
| UT-029..031, UT-065 | `internal/extension/manager_env_resolution_test.go` (extend existing) |
| UT-032..037, UT-066 | `internal/extension/registry_network_test.go` (new) + `internal/extension/manifest_network_participation` suite (extend) |
| UT-067 | `internal/daemon/extensions_test.go` (actor binding happens at the service/dispatcher seam) |
| UT-038..042, UT-069 | `internal/extension/inventory_test.go` (new) |
| UT-070 | `internal/api/core/extensions_test.go` (contract/action-result shape) |
| UT-043..046 | `internal/extension/build_test.go` + `manifest_load` suites (extend) |
| UT-047..051 | `internal/cli/extension_test.go` + `extension_output_test.go` goldens (extend) |
| UT-052..055 | `internal/api/core/extensions_test.go` + `payload_helpers_test.go` (extend; bundle cases removed in the same change) |
| UT-056 | `internal/skills` resource-spec/payload suites (extend) |
| UT-057 | `internal/extension/surfaces` + `registry_lifecycle` suites (extend) |
| UT-058 | `internal/tools/builtin/builtin_test.go` + `testdata/native-tool-catalog.json` golden + `internal/toolmeta/registry_test.go` (extend) |
| UT-060..061 | `internal/extension/events_coverage_test.go` + emitting-site tests (extend) |
| UT-062 | `web/src/systems/marketplace` existing kind/component suites (modify) |
| UT-063..064 | `web/src/systems/extensions` component/hook suites (extend) |
| IT-001..002 | `internal/store/globaldb` canonical migration suites (fresh/reopen/ahead/integrity/equivalence) |
| IT-003..006 | `internal/daemon/agent_skill_resources_integration_test.go`, new `automation_resources_integration` + `layout_resources_integration` suites (co-located with providers), `internal/daemon/extensions_test.go` |
| IT-007..008, IT-015 | `internal/daemon/extension_secrets_integration_test.go` (new) + doctor suite (`internal/api/core/extension_doctor` tests) |
| IT-009..010, IT-014 | `internal/daemon/extensions_test.go` + `internal/extension/marketplace_update` suites (extend) |
| IT-017 | `internal/daemon/agent_skill_resources_integration_test.go` + per-kind provider integration suites (seeded-row convergence) |
| IT-018 | `internal/daemon/extension_secrets_integration_test.go` + `extensions_test.go` (cross-workspace isolation) |
| IT-019 | `internal/daemon/extensions_test.go` (lifecycle coordinator concurrency/rollback) |
| IT-020 | `internal/daemon/extension_secrets_integration_test.go` (binding retirement) |
| IT-011 | `internal/api/httpapi/handlers_test.go`, `internal/api/udsapi/handlers_test.go`, `transport_parity_integration_test.go`, `internal/api/spec` operation tests (modify) |
| IT-012 | `internal/daemon/native_tools_test.go` (extend; bundle cases removed) |
| IT-013 | `internal/daemon/agent_skill_resources_integration_test.go` + reserved-name suite (extend) |
| IT-016 | existing `internal/api/core/support_test.go` + skills source-precedence suites (unchanged — cited as gate) |
| E2E-001..003, E2E-006 | Go E2E runtime harness suites under `internal/daemon/*_e2e_integration_test.go` (new kit journey file; dev-cycle journey extends the existing one) |
| E2E-004..005 | `web/e2e/__tests__/marketplace.spec.ts` + extension detail spec (modify/replace) |
| Site truth | `packages/site/lib/__tests__/{runtime-docs-discovery,runtime-tools-canonical-docs,runtime-docs-truth}.test.ts` (modify to post-cut truth) |

## Verification Commands

Per-phase gates (TechSpec Development Sequencing): scoped `go test -race ./internal/extension/... ./internal/daemon/...` + `make lint`; `make codegen && make codegen-check` after every contract/schema phase; `bunx turbo run lint typecheck test --filter=./web` for phase E; `make test-e2e-runtime` for the kit journeys; `make test-e2e-web` for Playwright; one full `make verify` at program completion; the living-reference grep gate (`/api/bundles`, `compozy bundle `, `compozy__bundles_`, `MarketplaceKindBundle`) expecting only historical/homonym hits.
