# Test Specification: Opt-In Agent Network Participation

Canonical test contract for the Local + Live participation redesign. Companion to `_techspec.md`. Derived from `_user_stories.md` (behavior; US-013 withdrawn) and `_techspec.md` (components). Every ID is permanent; extend existing canonical suites — never create parallel duplicate suites (`eng-consolidate-test-suites`).

## Strategy

- Go unit: table-driven, `t.Run("Should …")` + `t.Parallel` default, `-race`; fakes only at I/O boundaries (stub prompt runner, fake clock for coalescing/deadlines, in-memory SQLite per test).
- Go integration (`+integration` tag): real wiring across store/service/daemon composition in the existing manager/daemon integration suites; migration tests cover fresh-DB and reopen-after-restart (L-008).
- Runtime E2E (`make test-e2e-runtime`): daemon harness against `acpmock`; runtime-contract co-ship (L-007) — mock + matchers updated with the participation payloads in the same change.
- Web: vitest component/adapter suites for controls/invitation/empty states; Playwright (`make test-e2e-web`) for journeys. UI verification evidence via `eng-ui-screenshot` at task completion (not pixel tests).
- Status-code assertions always pair with body/contract assertions; deterministic time and IDs; no discarded errors.

## Coverage Matrix

| Source | Behavior | Unit | Integration | E2E |
|---|---|---|---|---|
| US-001 | Plain session fully local | UT-011 | IT-007 | E2E-001 |
| US-001.EC-1 | Admin-disabled ⇒ identical local | UT-014 | IT-008 | — |
| US-001.EC-2 | Explicit local records explicit source | UT-012 | — | — |
| US-001.EC-3 | Restart never acquires participation | — | IT-009 | — |
| US-001.EC-4 | Many local sessions ⇒ zero artifacts | — | IT-007 | E2E-001 |
| US-002 | Plain task run ⇒ no coordination artifacts | UT-013 | IT-004 | E2E-001 |
| US-002.EC-1 | Retry stays local | — | IT-005 | — |
| US-002.EC-2 | Recovery preserves local | — | IT-005 | — |
| US-002.EC-3 | Pre-cut artifacts reset by migration | — | IT-002 | — |
| US-003 | Plain loop run local for all steps | — | IT-017 | E2E-001 |
| US-003.EC-1 | High-volume loop ⇒ zero artifacts | — | IT-017 | — |
| US-003.EC-2 | Copied local template stays local | UT-047 | — | — |
| US-004 | Automation never enrolls implicitly | — | IT-020 | — |
| US-004.EC-1 | Webhook participation injection rejected | UT-016 | IT-021 | — |
| US-004.EC-2 | Duplicate fires resolve identically | — | IT-022 | — |
| US-005 | Loop-scoped participation | UT-015 | IT-018 | E2E-009 |
| US-005.EC-1 | Network step in local loop rejected at validate | UT-048 | IT-019 | E2E-009 |
| US-005.EC-2 | Concurrent runs get separate conversations | — | IT-018 | — |
| US-005.EC-3 | Unknown named channel fails, never creates | UT-017 | IT-028 | — |
| US-006 | Per-execution override | UT-018 | IT-006 | — |
| US-006.EC-1 | Channel outside authority ⇒ denied, no partial state | UT-019 | IT-006 | — |
| US-006.EC-2 | Double submission ⇒ one resolution | — | IT-006 | — |
| US-007 | No inheritance (children/review/detached) | UT-020 | IT-010 | — |
| US-007.EC-1 | Child within authority allowed, own source | UT-020 | IT-010 | — |
| US-007.EC-2 | Deep chains resolve independently | — | IT-010 | — |
| US-007.EC-3 | Reviewer defaults local | — | IT-010 | — |
| US-008 | Mode semantics + no silent degradation | UT-001, UT-014 | IT-028 | E2E-005 |
| US-008.EC-1 | Live unsupported ⇒ typed reason | UT-021 | — | — |
| US-008.EC-2 | Saved intent vs later unavailability fails typed | — | IT-008 | — |
| US-009 | Workspace/task coordination opt-in | UT-051, UT-052 | IT-011 | E2E-002 |
| US-009.EC-1 | Enable while admin-disabled ⇒ explained | UT-053 | IT-029 | — |
| US-009.EC-2 | Concurrent toggles ⇒ last write wins | UT-054 | — | — |
| US-009.EC-3 | Setting inspectable with provenance | UT-051 | IT-029 | — |
| US-010 | Kanban invitation | UT-055 | — | E2E-003 |
| US-010.EC-1 | Admin-disabled ⇒ no invitation | UT-055 | — | — |
| US-010.EC-2 | Single-agent run ⇒ no invitation | UT-055 | — | — |
| US-010.EC-3 | Double-accept ⇒ one enablement | UT-056 | — | E2E-003 |
| US-010.EC-4 | Run finishes with invite open ⇒ graceful | UT-056 | — | — |
| US-011 | Conversation visible; state authoritative | — | IT-012 | E2E-002 |
| US-011.EC-1 | Empty conversation normal | UT-057 | — | E2E-002 |
| US-011.EC-2 | Chatty run stays responsive (pagination) | UT-057 | — | — |
| US-011.EC-3 | Post-terminal message is history only | — | IT-013 | — |
| US-012 | Orchestration independent of network | — | IT-014 | E2E-001 |
| US-012.EC-1 | Network failure mid-run ⇒ run completes | — | IT-015 | — |
| US-012.EC-2 | Coordination on, no messages ⇒ zero wakes | — | IT-012 | — |
| US-013 | (withdrawn — mailbox out of scope) | — | — | — |
| US-014 | Live wake within visible bounds | UT-027–UT-034 | IT-023 | E2E-006 |
| US-014.EC-1 | Burst coalesces to one wake | UT-030 | IT-024 | — |
| US-014.EC-2 | Two live agents ⇒ depth-bounded | UT-031 | IT-024 | — |
| US-014.EC-3 | Cancel mid-wake charged; no re-wake | UT-035 | IT-025 | — |
| US-014.EC-4 | Boundary arrival wakes-or-accumulates atomically | UT-032 | — | — |
| US-015 | Always-visible area + oriented empty state | UT-058 | — | E2E-004 |
| US-015.EC-1 | Disabled state explained | UT-058 | — | E2E-005 |
| US-015.EC-2 | Empty→content transition clean | — | — | E2E-004 |
| US-015.EC-3 | Return to empty state graceful | UT-058 | — | — |
| US-016 | Onboarding mention enables nothing | UT-059 | — | E2E-004 |
| US-016.EC-1 | Dismissing onboarding ⇒ no network effect | UT-059 | — | — |
| US-017 | Truthful attributable usage | UT-036–UT-039 | IT-026 | E2E-002 |
| US-017.EC-1 | Workspace-scoped usage reads | — | IT-026 | — |
| US-017.EC-2 | Aggregates consistent with detail | UT-039 | IT-026 | — |
| US-017.EC-3 | Cancelled work's portion appears | UT-035 | — | — |
| US-018 | Availability off/on, data preserved | — | IT-030 | E2E-005 |
| US-018.EC-1 | Disable during live ⇒ clean stop, state kept | — | IT-030 | — |
| US-018.EC-2 | Non-admin toggle denied | — | IT-030 | — |
| US-018.EC-3 | Rapid toggle ⇒ no dup/lost state | — | IT-030 | — |
| US-019 | Agent-manageable via structured surfaces | — | IT-031 | E2E-007 |
| US-019.EC-1 | Local agent gets not_participating diagnostic | — | IT-016 | E2E-007 |
| US-019.EC-2 | Agent self-widening denied | UT-019 | IT-031 | — |
| US-020 | Definitions/extensions declare explicitly | — | IT-033 | — |
| US-020.EC-1 | Imported definition inert until confirmed | — | IT-033 | — |
| US-020.EC-2 | Extension update requirement re-confirmed | — | IT-034 | — |
| Contract package (TechSpec: Core Interfaces) | Validation + diagnostics | UT-001–UT-010 | — | — |
| Resolver (TechSpec: Core Interfaces) | Precedence + derivation + gates | UT-011–UT-021 | IT-001 | — |
| Config lifecycle (TechSpec: Config Lifecycle) | New/removed keys | UT-022–UT-026 | — | — |
| Migration v82+ (TechSpec: Data Models) | Hard cut, fresh+reopen, C-01 | — | IT-001–IT-003 | — |
| Delivery restructure (TechSpec: invariants 20-21) | Accept tx, dispatch, work authority (C-02/04/05) | UT-041–UT-046 | IT-023, IT-027 | E2E-010 |
| Admission/budget (TechSpec: invariants 9-16) | Eligibility, dedup, coalesce, ledger (C-03) | UT-027–UT-040 | IT-024–IT-025 | E2E-006 |
| Session-keyed relations (ADR-007, invariant 22) | Restart continuity | — | IT-027 | — |
| Status projection (C-06, invariant 19) | Typed, zero-activation | — | IT-013 | — |
| API/CLI parity (TechSpec: Agent Manageability) | Codes + shapes everywhere | — | IT-028–IT-032 | E2E-007 |
| Hooks/extensibility (TechSpec: Extensibility) | Payload swap, no widen | — | IT-033–IT-035 | — |
| Manual exact-run claim (TechSpec: invariant 18; B-001/B-008) | `ClaimRun` deleted; existing `ClaimCriteria.RunID` via `ClaimNextRun` | UT-061, UT-062 | IT-036 | — |
| Admission concurrency (TechSpec: invariants 10-11; B-003) | No overdraw under real concurrent connections | — | IT-037 | — |
| Wake-as-task-run (TechSpec: invariant 17; ADR-011) | Enqueue, claim, recovery via task_runs | — | IT-038 | — |
| Availability gate (TechSpec: invariant 24; B-006) | Disable/admit linearization + re-enable idempotency | — | IT-039 | — |
| Security non-regression (TechSpec: invariants 25-26; B-005) | Token redaction + identity classification survive the rewrite | UT-063, UT-064 | IT-040 | — |

## Unit Tests

### Participation contract (TechSpec: Core Interfaces)

- **UT-001** (happy): `participation.Validate` — `{mode: live, channel_strategy: named, channel_id: "builders", bounds: valid}` passes and normalizes.
- **UT-002** (error): mode `local` with `channel_id` set → `ErrStrategyChannelConflict` (`network_strategy_channel_conflict`).
- **UT-003** (error): `live` without strategy → `ErrStrategyInvalid`; `named` without `channel_id` → `ErrStrategyInvalid`.
- **UT-004** (error): non-`named` strategy with `channel_id` → `ErrStrategyChannelConflict`.
- **UT-005** (error): `live` with zero/absent `max_wakes` → `ErrBoundsExceedCeiling` variant `bounds_required` (zero never means unlimited).
- **UT-006** (boundary): duration strings — `"250ms"` valid; `"0s"`, `"-1s"`, `"250"` invalid with field-named error.
- **UT-007** (error): `run` strategy without `RunID` in `ResolveInput` → `ErrStrategyInvalid` naming the owning kind.
- **UT-008** (happy): `Spec` serializes with `version: "network-participation/v1"` and round-trips losslessly.
- **UT-009** (error): unknown mode string → validation error listing allowed values.
- **UT-010** (state): zero-value `Request` resolves to `{mode: local, source: built_in_local}` with empty channel and no bounds.

### Resolver (TechSpec: Core Interfaces; invariants 1-8)

- **UT-011** (happy): no request, no definition → `local`/`built_in_local`; asserts zero channel lookups performed (stub store records calls).
- **UT-012** (happy): explicit `local` request → source `explicit_request` (distinguishable from default).
- **UT-013** (happy): task run with no intent anywhere → `local`; resolver never consults the workspace record for non-coordinated owners.
- **UT-014** (error): request `live` with availability off → `ErrUnavailable`; returned before any store write (stub asserts no writes).
- **UT-015** (happy): loop definition `live`+`loop_run` strategy → derived channel from `LoopRunID`, source `loop_definition`.
- **UT-016** (error): automation fire with payload-injected request while job schema does not authorize it → injection ignored, job definition wins (source `automation_job`).
- **UT-017** (error): `named` channel absent in workspace → `ErrChannelUnknown`; no channel row created.
- **UT-018** (happy): precedence — request `local` over profile `live` → `local`/`explicit_request`; request `live` over workspace-enabled → `live`/`explicit_request`.
- **UT-019** (error): child request channel outside delegated authority → `ErrAuthorityDenied` (`network_participation_permission_denied`).
- **UT-020** (happy): child/review/detached inputs with participating parent and nil own request → `local` (no inheritance).
- **UT-021** (error): `live` for a profile that cannot honor bounded-wake guarantees → `ErrLiveUnsupported` with `local` named as supported.

### Config lifecycle (TechSpec: Config Lifecycle)

- **UT-022** (happy): `[network.live.defaults]` fills omitted bounds only when mode resolved `live`; a `local` resolution reads no defaults.
- **UT-023** (error): default above its `[network.live.limits]` ceiling fails load validation naming both keys.
- **UT-024** (error): removed keys (`network.default_channel`, `network.activation_top_k`, `network.port`) present → strict validation failure (no compatibility read).
- **UT-025** (boundary): request bound above ceiling → `ErrBoundsExceedCeiling`; equal to ceiling passes (inclusive).
- **UT-026** (happy): `agh config` tool-surface paths expose the new keys; removed paths gone.

### Live admission and budget ledger (TechSpec: invariants 9-16; ADR-008/ADR-011)

- **UT-027** (happy): direct message to live participant with budget → within the acceptance transaction, `Admit` enqueues a `task_runs` record (`run_kind='network_wake'`, `task_id` NULL, targeted session) and writes ledger/sources/counters atomically with the conversation row; returned reservation carries `TaskRunID` + `OwnerKey`.
- **UT-028** (error): `greet`/`whois`/`receipt`/`trace`/status-projection kinds → structurally ineligible before admission (skip reason `not_eligible`), zero reservations.
- **UT-029** (happy): mention resolved to participant admits; unaddressed broadcast to live participant does NOT (skip `not_addressed` — no capability auto-activation).
- **UT-030** (concurrency): 10 envelopes within the coalesce window (fake clock) → exactly one open wake for the owner (`UNIQUE(owner_key) WHERE state='open'`); all 10 persisted as attached sources; an envelope arriving after claim or `coalesce_until` accumulates without admitting.
- **UT-031** (boundary): causation depth at `max_wake_depth` → skip `depth_exceeded`; message accumulates.
- **UT-032** (boundary): last remaining wake + concurrent arrivals → exactly one admitted, other accumulates (`budget_exhausted`), never half-processed.
- **UT-033** (idempotency): same envelope re-presented after settle, after restart, and after availability re-enable → no second admission (`network_wake_sources` primary key is the permanent idempotency record).
- **UT-034** (state): exhausted participation → subsequent eligible envelopes skip with `budget_exhausted` + visible reason on participation read.
- **UT-035** (state): cancel mid-wake → `Settle` records `canceled` with consumed portion; same envelope cannot re-admit.

### Settlement and usage (TechSpec: invariant 15; C-03)

- **UT-036** (happy): terminal success with usage → ledger `input/output_tokens` actual, `usage_state=actual`, budget decremented by actuals.
- **UT-037** (error): terminal error event → outcome `failed`, never `delivered`; reservation stays consumed.
- **UT-038** (error): missing usage → `usage_state=usage_unavailable`; conservative reservation kept; no estimate stored as actual.
- **UT-039** (happy): usage query by run/channel/workspace sums ledger rows and equals per-wake detail; zero-participation workspace returns zero.
- **UT-040** (boundary): wall-time deadline (fake clock) fires mid-turn → turn cancel requested; settle as `deadline_exceeded`, charged.

### Delivery accept + dispatch (TechSpec: invariants 20-21; ADR-010)

- **UT-041** (happy): accept transaction persists conversation row + routing decision computed from pre-message participant snapshot (first-send selects existing recipients — C-02 regression).
- **UT-042** (idempotency): same message ID re-sent → durable state returned; no duplicate row, no phantom success semantics (C-05 regression).
- **UT-043** (error): accept transaction failure → nothing persisted, typed error to sender.
- **UT-044** (happy): dispatcher consumes persisted decision; recipients receive exactly the committed set.
- **UT-045** (state): `network_work` transitions valid only through the store repository; no in-memory authority (C-04 regression — restart-shaped unit with reopened store).
- **UT-046** (ordering): per-recipient dispatch preserves acceptance order.

### Loop validation (TechSpec: invariant 7)

- **UT-047** (happy): local loop definition with no network nodes validates; copied definition stays local.
- **UT-048** (error): local loop with `agh__network_send`/`channel_result` node → `loop_requires_live` naming the node; dry-run reports the same item.
- **UT-049** (happy): live loop → nodes inherit the one loop-run conversation; no per-node channels.
- **UT-050** (error): live loop start when availability off → `network_participation_unavailable` before run creation.

### Workspace coordination service (ADR-009)

- **UT-051** (happy): `Set(true, actor)` then `Get` → enabled with `updated_by`/`updated_at` provenance.
- **UT-052** (happy): task-profile intent overrides workspace record for that task (precedence, invariant 2).
- **UT-053** (error): `Set` while availability off → typed unavailable; record unchanged.
- **UT-054** (concurrency): two `Set` calls with controlled transaction order → the later commit wins per the monotonic `updated_at`/revision tie-break contract; both readers observe that explicit winner (deterministic oracle, N-004).

### Web units (vitest, existing web suites)

- **UT-055** (state): invitation trigger — renders only when run active + coordinator + ≥2 workers + scope off + network available; each condition false ⇒ hidden.
- **UT-056** (idempotency): invitation accept double-click → one PUT; dismissal issues the invitation PUT and renders from the daemon-returned per-scope state (never browser-local storage); invitation copy states acceptance affects future runs only; open invite resolves gracefully on run completion.
- **UT-057** (state): run conversation panel — empty state explains silence; long transcript paginates; run view stays interactive.
- **UT-058** (state): network area empty states answer the four orientation questions with one action; disabled variant names who can change it; no fabricated data.
- **UT-059** (state): onboarding item links to the area and mutates no settings on visit/complete/dismiss.
- **UT-060** (happy): session/task/loop/automation participation controls serialize `network_participation` exactly per contract; legacy `channel` field absent from payloads.

### Manual exact-run claim (TechSpec: invariant 18; B-001)

- **UT-061** (happy): the existing `ClaimCriteria{RunID: "run-1"}` through `ClaimNextRun` → claims exactly run-1 with claim-token issuance and lease fields identical to criteria-based claims (no second exact selector exists; B-008).
- **UT-062** (error): `RunID` targeting a non-queued/leased/terminal run → typed claim failure; no state mutation, no silent fallback to criteria matching.

### Security non-regression (TechSpec: invariants 25-26; B-005)

- **UT-063** (error): accept-path validation — envelopes carrying raw `agh_claim_*` material in body, proof, ext, or nested metadata → rejected before any durable write; the typed error carries `claim_token_hash` only.
- **UT-064** (error): verified-format `nickname@fingerprint` identity with missing/invalid proof → classified `rejected` (never `unverified`); ordinary non-verified-format identity without proof → `unverified`.

## Integration Tests

### Migration and store (canonical: globaldb suites)

- **IT-001**: fresh DB at v82+ — new columns/tables present; reservation persists resolved snapshot atomically with the run row.
- **IT-002**: reopen-after-restart from a pre-cut fixture — destructive cut applied once: old channel columns gone, pre-cut auto-created `task_run_coordination` channels and peer-keyed relation rows reset; history readable without broken links; **every retained session/task-run/loop-run row's `network_spec_json` decodes as the canonical Local `Spec`** (version + mode + source) with matching projections — no empty or undecodable snapshot exists post-upgrade (B-012).
- **IT-003**: two workspaces, same channel name `ops` — every lookup/cache path returns only its workspace's row (C-01 regression, invariant 8).
- **IT-004**: plain workspace task start — zero `network_channels` mutations, zero conversation state, run snapshot `local`/`built_in_local` (replaces the frozen auto-channel expectations).
- **IT-005**: retry + crash-recovery of a local run — recovered attempt keeps `local`; no re-derivation.
- **IT-006**: run-start override — one run `live`/`explicit_request` while its task profile says local (and vice versa); double submission yields one resolution; authority-denied override leaves no partial rows.

### Session lifecycle and projection (canonical: session/daemon suites)

- **IT-007**: local session end-to-end — no `AGH_SESSION_CHANNEL`/`AGH_PEER_ID`, no join, no network prompt section, no coordination toolset in `SessionProjection`, nothing in network reads.
- **IT-008**: `network.enabled=false` — local sessions unchanged; live create fails `network_participation_unavailable` pre-persistence; saved live intent fails identically at next execution.
- **IT-009**: live session restart — same session id reattaches: direct rooms, subscriptions, thread membership preserved (invariant 22); snapshot unchanged.
- **IT-010**: spawn/review/detached from a live parent — children resolve `local`; explicit child request within authority joins with own source; outside authority → typed denial.
- **IT-011**: workspace coordination enabled → next coordinated run resolves `live`/`workspace_coordination`, gets run-derived conversation; disable → next run local; in-flight run keeps snapshot.
- **IT-012**: coordinated live run with zero messages → zero wakes, zero usage rows (US-012.EC-2); with messages → conversation projected on run detail read model.

### Coordinator, scheduler, status projection (canonical: coordinator/daemon suites)

- **IT-013**: typed task-status projection wired at normal boot order — real task transitions produce the typed projection with **zero observed `PromptNetwork` calls and zero conversational messages** (behavioral assertion via prompt-runner spy and timeline read; C-06); post-terminal message renders as history and mutates nothing (invariant 19).
- **IT-014**: coordinator bootstraps for a local run (no channel anywhere) — claims/reviews/terminal states function; allowlist contains task tools, no network trio; overlay has no channel guidance.
- **IT-015**: network failure injected mid-participating-run — task orchestration completes; conversation marked unavailable truthfully.
- **IT-016**: local agent session invokes a coordination tool → deterministic `not_participating` diagnostic naming the enabling surface.

### Loop and automation (canonical: loop/automation suites)

- **IT-017**: local loop run (multi-node) — zero channels/memberships for actions and judges; dry-run shows `local` explicitly.
- **IT-018**: live-declared loop — one conversation per loop run shared by nodes; two concurrent runs isolated.
- **IT-019**: loop with network node + resolved local → start and dry-run both fail `loop_requires_live`.
- **IT-020**: automation job (no declaration) fires task and loop targets → both local.
- **IT-021**: webhook fire with injected participation → ignored; fire record shows job-definition resolution.
- **IT-022**: retried automation fire → identical resolution both attempts.

### Delivery, admission, relations (canonical: network manager/delivery suites)

- **IT-023**: full send→accept→dispatch→admit→wake→settle through real wiring with `acpmock` — one direct message wakes the target once; ledger row settled with actual usage.
- **IT-024**: burst of 10 messages + two live agents replying — coalesced single wake per root; reply chain stops at `max_wake_depth`; all messages durable.
- **IT-025**: cancel during wake — provider cancel propagated (mock asserts), outcome `canceled`, charged, no re-wake on redelivery.
- **IT-026**: usage endpoint/CLI aggregation equals ledger detail across run/channel/workspace scopes; foreign-workspace query denied.
- **IT-027**: restart with an admitted-but-unexecuted wake — the wake `task_run` recovers through **normal task-run recovery** (no network-owned recovery path exists; dispositions are immutable evidence with no consumption or cursor state); re-presented envelopes cannot re-admit (`network_wake_sources`); work authority survives restart in SQLite only (ADR-011; B-018).

### API/CLI parity (canonical: api/core, httpapi, udsapi, cli suites)

- **IT-028**: session/task/loop/automation payloads with `network_participation` round-trip identically over HTTP and UDS; legacy `channel`/`network_channel`/`coordination_channel_id` fields → schema validation failure (`unknown_field`) with the field named.
- **IT-029**: `GET/PUT /api/workspaces/{id}/network-coordination` — read/write with provenance; PUT while disabled → `network_participation_unavailable` body + appropriate status; body asserted, not status-only.
- **IT-030**: availability toggle journey — disable: live create rejected, local work unaffected, network reads show disabled-with-data; enable: intact; rapid toggle: consistent; non-admin: denied.
- **IT-031**: CLI structured outputs (`session new --network live …`, `network coordination`, `network usage`, `network status`) match HTTP/UDS state field-for-field; error codes identical across surfaces.
- **IT-032**: `--network-bounds` structured input validates against the same contract (ceiling violation → same `network_bounds_exceed_ceiling` code as HTTP).
- **IT-033**: hook payloads carry the resolved participation object (no flat legacy fields); pre-resolution hook may deny/narrow a request; widen attempt without capability → denied.
- **IT-034**: extension/bundle declaring live requirement → activation preview surfaces it; the explicit confirmation field on the activate contract persists `NetworkRequirementDigest/ConfirmedBy/ConfirmedAt` **on the typed `bundle.activation` resource** (`ActivationResourceSpec` in `resource_records.spec_json`, fenced by the resource's expected-version CAS; the canonical suite keeps asserting no legacy activation table exists); decline → activation fails naming the requirement with no partial state; update changing the requirement block clears the confirmation fields in the same versioned resource update and re-confirms; same behavior through HTTP, UDS, and CLI `--confirm-network-requirement` (N-008/B-016/B-019).
- **IT-036**: manual exact-run claim journey — operator API/CLI claims a specific queued run via the existing `ClaimCriteria.RunID`: same claim token/lease/heartbeat/hooks/events as autonomous claims; the legacy direct-mutation claim behavior is gone from every public surface (B-001/B-008).
- **IT-037**: adversarial admission concurrency — N concurrent real connections admitting against one owner near its ceilings, across **distinct roots** → never more than one open wake per owner (`UNIQUE(owner_key) WHERE state='open'`), total admitted wakes never exceed remaining budget, every source envelope linked exactly once; claim-vs-coalesce race (source attach concurrent with run claim) resolves deterministically; duplicate terminal settlement no-ops (CAS); restart replay of the same terminal outcome cannot double-apply counters (B-003/B-011).
- **IT-038**: wake execution path — admitted wake run (`run_kind='network_wake'`, `task_id` NULL) is claimed via `task.Service.ClaimNextRun` by the `NetworkWakeRunner` for the target session, prompts once, settles; a foreign session cannot claim it; `tasks.current_run_id` and task status projections are untouched by the wake's whole lifecycle; daemon restart between enqueue and claim recovers it through standard run recovery with no duplicate admission (ADR-011/B-009).
- **IT-039**: availability linearization — concurrent `network.enabled=false` apply (which writes `network_availability` transactionally) and admission transactions → each admission commits strictly before the disable write or fails `network_disabled` and accumulates (asserted via the persisted epoch); in-flight wake receives cancel-request and settles truthfully; re-enable advances the epoch and does not re-admit settled/accumulated envelopes (B-006/B-013).
- **IT-040**: security sweep across surfaces — raw `agh_claim_*` injected via HTTP send, UDS send, CLI `ch send`, native `agh__network_send`, and extension host API → all rejected pre-persistence with hash-only diagnostics; emitted events/SSE/logs for the attempts contain no raw token; proof-stripped verified-format identity over each surface classifies `rejected` (B-005).
- **IT-035**: bundle default-channel deletion, behaviorally — a bundle with a primary channel activated in the workspace + a plain session create → the session resolves `local`/`built_in_local` with empty channel; no bundle-sourced participation exists through any public read (symbol removal is owned by compilation and the delete-target sweep, not this test).

## End-to-End Tests

### Light user stays local (US-001–US-004, US-012)

- **E2E-001** (runtime): fresh daemon, default config → create session, start task to terminal, run local loop → assert zero network rows, zero network env/tools in agent context (acpmock captures), zero wakes, zero usage; kanban orchestration fully functional.

### Coordinated-run flagship (US-009, US-011, US-017)

- **E2E-002** (runtime): enable workspace coordination via API → start coordinated run with coordinator + 2 workers → run detail exposes resolved `live`/`workspace_coordination` + conversation; messages exchanged; usage attributed to the run; task state transitions owned by task engine only.

### Invitation journey (US-010)

- **E2E-003** (web/Playwright): active multi-agent run with coordination off → invitation visible in run detail stating that acceptance affects future runs only → accept → confirmation + setting enabled (GET reflects) → invitation gone; dismiss path persists across reload and browser change (daemon-backed `network_coordination_invitations`, verified via GET).

### Discoverability (US-015, US-016)

- **E2E-004** (web): zero-participation install → network nav present → empty state shows orientation + one action → complete action → area shows real content; onboarding item visits without changing settings.

### Administration (US-008, US-018)

- **E2E-005** (runtime + web): disable availability → live create fails with named diagnostic (CLI + web show it), local flows unaffected, network area shows disabled state → re-enable → prior conversations intact.

### Live bounds (US-014)

- **E2E-006** (runtime): two live sessions in a channel → direct + mention wake within bounds; burst coalesces; exhaust `max_wakes` → further messages accumulate with visible `budget_exhausted`; transcript complete.

### Agent manageability (US-019)

- **E2E-007** (runtime): a managed agent session inspects its own participation, queries usage, attempts coordination enable within/without authority — structured outputs and deterministic codes throughout; local agent receives `not_participating` guidance.

### Loop participation (US-005)

- **E2E-009** (runtime): live-declared loop runs with one conversation; local loop with network node rejected at validate with `loop_requires_live`. *(E2E-008 intentionally unassigned — reserved gap, never reuse.)*

### Explicit network still works post-cut (US-006, regression)

- **E2E-010** (runtime): explicit live sessions on a named channel — thread reply, direct room, work lifecycle, task promotion all function under the new dispatcher (existing collaboration journey preserved).
