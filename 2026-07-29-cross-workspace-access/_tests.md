# Test Specification: Cross-Workspace Access via Session PermissionMode

Canonical test contract. Companion to `_techspec.md` (post-ADR-007 mode-anchored design).
No `_user_stories.md` exists; journeys derive from the in-session decisions in `adrs/adr-001..007.md`. IDs that earlier descopes (ADR-006) or the ADR-007 re-anchoring removed are marked `(withdrawn)` — never renumbered. Surviving IDs keep their number; their text is the current contract.

## Strategy

- Frameworks: Go `testing` (`t.Run("Should …")` + `t.Parallel`, `-race`); fakes only at I/O boundaries (fake `ModeSource`/`SessionConsentCache`/`AuditEmitter`/prompter; `acpmock` for runtime E2E); real daemon session state in integration; Vitest for web units; Playwright for web E2E through the daemon.
- Execution: unit `go test -race ./internal/<pkg>/...` + `bunx turbo run test --filter=./web`; integration `make test-integration`; runtime E2E `make test-e2e-runtime`; web E2E `make test-e2e-web`; completion gate `make verify`.
- **Suite ownership (enforced):** the policy decision chain lives **only** in the `workspaceaccess` unit suite. Each integration case proves exactly one substrate boundary using the minimum representative decision (never the mode matrix). E2E proves cross-surface journeys only. Regression floor: existing isolation suites pass with wiring-only edits plus the named deltas of safety invariant 1.

## Coverage Matrix

| Source | Behavior | Unit | Integration | E2E |
| --- | --- | --- | --- | --- |
| ADR-001/007 | Decision chain (step-zero validation, order, kind gate, mode mapping, fail-closed) | UT-001, UT-002, UT-007, UT-011, UT-051, UT-053, UT-063, UT-076–UT-082 | — (unit-owned) | — |
| Audit (inv. 10) | One best-effort record per decision | UT-012, UT-013, UT-055 | IT-003 | — |
| Mode provenance | Daemon `ModeSource` over live session state | — | IT-024 | — |
| Session consent | Prompt answers, cache lifecycle, cross-seam validity | UT-033, UT-036, UT-083, UT-084 | IT-011, IT-025 | E2E-002–E2E-004, E2E-013 |
| Identity seam | Agent callers cross-workspace | UT-021–UT-023 | IT-009 | E2E-006 |
| Task seams | Authorizer, run-read mask, claims | UT-024–UT-028 | IT-010 | E2E-007 |
| Tool seam + prompt | Binder consult, reason code, prompt containment | UT-029–UT-032, UT-085 | IT-011 | E2E-001–E2E-005 |
| Spawn / Coordination | Policy consults, double-validation audit | UT-037–UT-039 | — | E2E-007, E2E-013 |
| Events registry | Types + notify classes | UT-046 | IT-003 | — |
| ADR-004 (phase 0) | Owner projection minimality + confirm + cache isolation | UT-047–UT-049, UT-061, UT-062 | IT-013, IT-020 | E2E-008, E2E-009 |
| CLI pre-flight delete | Daemon-origin denial | UT-043 | — | E2E-006 |
| Regression | Pre-feature outcomes + named deltas only | — | IT-016 | E2E-011 |

## Unit Tests

### `internal/workspaceaccess` — DefaultPolicy (unit-owned decision chain)

- **UT-001** (happy): `Operator=true`, foreign target — `{Allowed:true, Source:"operator"}`; mode source and consent cache untouched.
- **UT-002** (happy): agent_session, home `ws_a`, target `ws_a` — `{Allowed:true, Source:"same_workspace"}`; mode source untouched.
- **UT-003–UT-006** *(withdrawn — grant/toggle chain removed by ADR-007; mode mapping owns the chain in UT-076–UT-080)*.
- **UT-007** (error): empty `TargetWorkspaceID` — error deny at step zero.
- **UT-008/UT-009** *(withdrawn — workspace-less toggle branches removed by ADR-007; UT-082 owns workspace-less behavior)*.
- **UT-010** *(withdrawn — grant-store precedence removed by ADR-007)*.
- **UT-011** (error): `New(Deps)` with nil `Modes`/`Consent`/`Audit` — constructor error each time.
- **UT-012** (state): allow outcome emits one `AccessRecord{Allowed:true}`; an emitter error is logged `Warn` and the decision is unchanged (best-effort).
- **UT-013** (state): deny outcome emits `AccessRecord{Allowed:false}`; emitter error likewise logged, decision unchanged.
- **UT-051** (state): kind gate — `extension`/`automation`/`network_peer`/`daemon` (non-operator), foreign target, mode `approve-all` on the referenced session → denied; only `agent_session` passes.
- **UT-052** *(withdrawn — ToggleSource removed by ADR-007; mode-source failure is UT-081)*.
- **UT-053** (boundary): non-canonical target (name/path ref) — error deny at step zero.
- **UT-055** (state): one `Authorize` call = exactly one emitted record, including error denials.
- **UT-063** (error): step-zero validation — unknown `ActorKind`, missing seam with a target, or empty `SessionID` on an agent-session actor — error deny before any branch.
- **UT-076** (happy): mode `approve-all`, foreign target — `{Allowed:true, Source:"permission_mode"}`; consent cache never consulted.
- **UT-077** (state): mode `deny-all`, foreign target — denied, `PromptEligible:false`, nil error; consent cache never consulted (hard deny, invariant 7).
- **UT-078** (state): mode `approve-reads`, consent miss — denied, `PromptEligible:true`, nil error.
- **UT-079** (happy): mode `approve-reads`, cached `ConsentAllow` — `{Allowed:true, Source:"session_consent"}`.
- **UT-080** (state): mode `approve-reads`, cached `ConsentReject` — denied, `PromptEligible:false`.
- **UT-081** (error): `ModeSource` error, unknown session, or unrecognized mode value — error deny; `AccessRecord.Err` and `Mode` populated accordingly.
- **UT-082** (boundary): workspace-less agent_session (home `""`) — same-workspace never matches; `approve-all` allows, `approve-reads` miss is `PromptEligible`, `deny-all` hard-denies.

### `internal/workspaceaccess` — Grant domain

- **UT-014/UT-015/UT-064** *(withdrawn — grant domain removed by ADR-007)*.

### `internal/config` — `[workspace_access]` *(section withdrawn)*

- **UT-016–UT-020** *(withdrawn — no config section exists after ADR-007; `permissions.*` trust-root protection is pre-existing behavior owned by the config suite)*.

### `internal/agentidentity` (seam)

- **UT-021** (error): nil policy + mismatch — `ErrIdentityUnauthorized`, exit 77 intact.
- **UT-022** (happy): policy allow — `Resolve` succeeds; home workspace not rewritten; `ActorRef` carried `Kind: agent_session` + snapshot session id.
- **UT-023** (error): policy deny — `ErrIdentityUnauthorized` with the mode-hint copy; a `PromptEligible` deny at this seam is a plain deny (no prompt machinery reachable).

### `internal/task` (seam)

- **UT-024** (happy): `AuthorizeTaskScope` non-operator agent_session, foreign, policy allow — nil.
- **UT-025** (error): policy deny — `ErrPermissionDenied`.
- **UT-026** (state): `AuthorizeRunRead` deny — `ErrTaskRunNotFound` mask preserved; audit emitted inside the policy.
- **UT-027** (happy): claim criteria foreign + allow — normalized to the requested workspace.
- **UT-028** (state): operator short-circuits before the policy (zero calls).
- **UT-054** *(withdrawn — the `CallerScope.AgentName` hard cut was removed by ADR-007; `ActorRef.AgentName` is best-effort audit attribution only)*.

### `internal/tools` + `internal/daemon` binder + prompt (seam)

- **UT-029** (happy): empty `workspace_id` input inherits trusted scope (fill-if-absent unchanged).
- **UT-030** (happy): mismatch + policy allow — proceeds with the requested workspace.
- **UT-031** (error): mismatch + policy deny, no live session — `ReasonWorkspaceAccessDenied` (not `ReasonScopeMismatch`); no prompt attempted.
- **UT-032** (error): `session_id` mismatch — `ReasonScopeMismatch`; policy never consulted.
- **UT-033** (happy): `PromptEligible` deny + live session + prompt `allow_once` — call proceeds; nothing cached (immediate repeat is `PromptEligible` again).
- **UT-034/UT-035** *(withdrawn — durable grant upserts removed by ADR-007; session-scoped consent is UT-083)*.
- **UT-036** (error): prompt timeout/error or out-of-set answer — denied, nothing cached.
- **UT-056–UT-058, UT-068–UT-070** *(withdrawn — flight keys, dispatch-id permits, outcome memos, and leader/follower/CAS race cases removed by ADR-006)*.
- **UT-083** (state): prompt `allow_session` — consent cached + call proceeds; prompt `reject_session` — reject cached + call denied; subsequent policy calls resolve from cache with zero prompts.
- **UT-084** (state): session stop clears the session's cached consent (daemon cleanup hook); a later session with the same agent name starts with an empty cache entry.
- **UT-085** (state): `PromptEligible` denials at non-tool seams (identity/task/spawn/coordination call sites) are handled as plain denies — no prompt requester is ever invoked.

### `internal/session` spawn + `internal/workspace` coordination (seams)

- **UT-037** (happy): spawn foreign + parent-session policy allow — child bound to the foreign workspace.
- **UT-038** (state): pre-create hook mutates spawn workspace — second validation consults the policy again (two calls, two audit records).
- **UT-039** (happy/error pair): coordination foreign read + allow — nil; foreign write without operator — denied regardless.

### `internal/api/core` handlers

- **UT-040** *(withdrawn — workspace-access status/grant routes removed by ADR-007)*.
- **UT-059/UT-060/UT-071/UT-072** *(withdrawn — operator token/presence, nonce exchange, and split status contracts removed by ADR-006)*.
- **UT-061** (happy): owner projection returns exactly `{session_id, workspace_id, workspace_name}`; unknown session → 404.

### `internal/cli`

- **UT-041/UT-042** *(withdrawn — `workspace access` CLI verbs removed by ADR-007)*.
- **UT-043** (state): `TestWorkspaceResolutionBoundary` extended — no required workspace flag; ID/name/path accepted; cross-workspace refs resolve locally and the daemon decides (behavioral; no tombstone assertions).
- **UT-073** *(withdrawn — obsolete since ADR-006)*.

### `internal/daemon` native tools + instructions

- **UT-044/UT-045** *(withdrawn — `compozy__workspace_access` toolset removed by ADR-007)*.
- **UT-075** *(withdrawn — the dedicated system-prompt rule was removed by ADR-007)*.

### `internal/events`

- **UT-046** (state): registry declares `workspace.access_denied` notify-eligible warning and `workspace.access_granted` success non-notify; no grant transition types exist.

### `web/` (Vitest)

- **UT-047** (happy): loader on active-workspace miss calls the owner projection (not the catalog) and returns dialog state carrying only `{sessionId, workspaceId, workspaceName}`.
- **UT-048** (state): projection 404 — today's not-found state, no dialog.
- **UT-049** (state): confirm switches the active-workspace store then navigates; cancel leaves it untouched.
- **UT-050** *(withdrawn — Settings workspace-access section removed by ADR-007)*.
- **UT-062** (state): owner response cached only under the `session-owner` key; no foreign catalog/detail entries pre-confirm.
- **UT-066/UT-067/UT-074** *(withdrawn — resolver interface and nonce boot flow removed by ADR-006)*.

## Integration Tests

### Real substrate boundaries (one per case)

- **IT-001/IT-002** *(withdrawn — grant store removed by ADR-007)*.
- **IT-003**: representative allow, deny, and error decisions each append exactly one queryable `EventSummary` (`Type`, actor-home `WorkspaceID`, payload target/seam/source/mode); a forced append failure leaves the decision unchanged and logs.
- **IT-024**: real `ModeSource` — the daemon resolves `EffectivePermissions` from a live managed session for the policy; an unknown/stopped session id denies with a wrapped error (fail-closed, invariant 2).
- **IT-025**: consent lifecycle through the daemon — `allow_session` recorded for a live session is visible to a subsequent `Authorize`; stopping the session clears it (next lookup is a miss).

### Grant store lifecycle

- **IT-004–IT-007, IT-017, IT-021** *(withdrawn — no grant store after ADR-007)*.

### Config + schema

- **IT-008** *(withdrawn — no config key after ADR-007)*.
- **IT-016**: regression floor — existing isolation suites green with an explicit `approve-reads` mode and no prompt answers: wiring-only edits plus the named deltas of safety invariant 1. No schema/migration assertions (nothing changed).

### Transport + seams (single responsibility each)

- **IT-009**: HTTP identity mapping — one representative denied-then-allowed pair (explicit `approve-reads` session → 403 + mode hint; an `approve-all` session → 200) proving header→snapshot→policy wiring; UDS mirror for the denied case. No further mode permutations.
- **IT-010**: claim propagation — one `approve-all` cross-workspace `task next` claim binds the foreign workspace end-to-end; one automation-actor request stays denied (kind gate at a real seam). Nothing else re-asserted.
- **IT-011**: binder + bridge + `acpmock` — `allow_session` answer feeds the consent cache and the session's next foreign call skips the prompt.
- **IT-012** *(withdrawn — grant routes removed by ADR-007)*.
- **IT-013**: `make codegen-check` clean after the owner-projection contract addition.
- **IT-014/IT-015** *(withdrawn — CLI verbs and native mutation tools removed by ADR-007)*.
- **IT-020**: owner projection parity — HTTP and UDS byte-identical minimal payloads; no agent/provider/activity fields present.
- **IT-018/IT-019/IT-022/IT-023** *(withdrawn — flight coalescing and operator-presence machinery removed by ADR-006)*.

## End-to-End Tests

### Agent hits the boundary at the tool seam (`acpmock`)

- **E2E-001**: `deny-all` session — foreign native tool call denied with `ReasonWorkspaceAccessDenied` + hint, **zero prompts**; audit deny recorded (invariant 7).
- **E2E-002**: `approve-reads` session, prompt → `allow_once` — call succeeds; immediate second call prompts again; nothing cached.
- **E2E-003**: prompt → `allow_session` — succeeds; third call succeeds promptless; a **new session for the same agent** prompts again (consent died with the session).
- **E2E-004**: prompt → `reject_session` — denied; subsequent calls in the session denied with zero prompts.
- **E2E-005**: `approve-all` session — foreign call succeeds promptless (journey only; the mode chain is unit-owned in UT-076–UT-080).

### Cross-seam parity

- **E2E-006**: agent-driven CLI inside an `approve-reads` session targeting `ws_b`: exit 77 + mode hint, daemon-origin, no prompt (identity seam never prompts); success inside an `approve-all` session; provenance reports `cross_workspace_attempt`.
- **E2E-007**: spawn child into `ws_b`: denied from an `approve-reads` parent (no prompt at the spawn seam); succeeds from an `approve-all` parent; child home = `ws_b`.
- **E2E-012** *(withdrawn — the dedicated system-prompt rule was removed by ADR-007)*.
- **E2E-013**: cross-seam consent reuse — in an `approve-reads` session, `allow_session` granted at the tool seam lets a subsequent cross-workspace task claim (or spawn) in the same session proceed without further consent; audit shows `Source: session_consent` at the second seam.

### Operator deep link (Playwright)

- **E2E-008**: two workspaces, `ws_a` active — both permalink forms for a `ws_b` session show the confirm dialog naming `ws_b`; confirm switches and opens; network log shows only the owner projection pre-confirm and no foreign payload in any cache.
- **E2E-009**: cancel keeps `ws_a` + arrangement, shows not-found; nonexistent session → not-found, no dialog.

### Settings management (Playwright)

- **E2E-010** *(withdrawn — no Settings surface after ADR-007)*.

### Regression journey

- **E2E-011**: existing web isolation journey (per the rewritten ADR-004 contract) plus the native foreign-workspace matrix with the default mode and no prompt answers — outcomes/status/sentinels identical to pre-feature behavior except the named deltas of invariant 1, each asserted explicitly.
