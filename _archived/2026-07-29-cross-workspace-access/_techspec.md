# TechSpec: Cross-Workspace Access via Session PermissionMode

No PRD exists; requirements were resolved in-session on 2026-07-28 and are recorded in `adrs/adr-001..007.md`. Three peer-review rounds ran (`qa/peer-review-findings-round1..3.md`) against the earlier grants-based design; **ADR-007 re-anchored the feature on the existing session `PermissionMode`** after Pedro's simplification decision, deleting the toggle/grants program (ADR-002/003 superseded). This document is the normative spec for the re-anchored design.

## Executive Summary

Workspace isolation for agent actors is currently a hard deny with no door. This spec opens the door using a model the operator already understands — the session permission mode:

| Session mode | Cross-workspace outcome |
| --- | --- |
| `approve-all` | allow |
| `deny-all` | hard deny — no prompt |
| `approve-reads` | ask: ACP prompt at the native-tool seam; deny + hint everywhere else |

One new package, `internal/workspaceaccess`, owns the decision; the existing enforcement seams consult it instead of denying inline (ADR-001, unchanged — the anti-drift funnel L-033 demands). The decision source is the session's persisted `EffectivePermissions` — **no new config key, no new table, no new HTTP/CLI/native-tool/Settings surface**. Prompt consent in `approve-reads` is session-scoped and in-memory (`allow_session`/`reject_session`), dying with the session. Every decision emits a best-effort audit event. Separately, phase 0 fixes the operator UX dead-end: deep links to foreign-workspace sessions resolve a minimal owner projection and ask before switching the active workspace (ADR-004, unchanged).

Trade-offs, stated plainly: tool permissiveness and the workspace boundary are now one axis — an `approve-all` session crosses freely, and the "tools open but workspace isolated" configuration no longer exists (ADR-007, accepted). `deny-all` means "never cross" on this axis even though it means "ask everything" for tool risk (deliberate lockdown semantics). The beta security posture of ADR-006 stands: a deliberately malicious agent editing config files on disk is detected (audit), not prevented; the hard boundary remains a named future security ADR.

**MVP Boundary:** phases 0 (owner projection + deep-link confirm) and 1 (policy + seam wiring + prompt + consent cache + audit) ship as one program; the phase-0 web track is parallelizable from day one. Out of scope permanently: workspace-pair trust lists, `read`/`full` capability levels, durable cross-workspace grants (the ADR-002/003 program). Out of scope for this spec: OS-enforced session containment (ADR-005 remains the design record), extension Host API cross-workspace ceiling, network-peer cross-workspace participation, authorization metrics.

## System Architecture

### Component Overview

- **`internal/workspaceaccess` (new)** — owns `Policy` (step-zero validation + fixed decision chain), the `ModeSource`/`SessionConsentCache`/`AuditEmitter` interfaces, and the audit record contract. No prompt logic, no transport, no session-manager knowledge, no config parsing.
- **Enforcement seams (modified)** — `internal/agentidentity`, `internal/task` (resource authorizer, claim hooks, taskless run read), `internal/tools` dispatch + `internal/daemon` native workspace binder, `internal/session` spawn, `internal/workspace` coordination. Each consults the injected policy on workspace mismatch for agent actors.
- **`internal/daemon` (modified)** — composition root: implements `ModeSource` over the session manager (`EffectivePermissions` lookup), owns the in-memory consent cache and its session-stop cleanup, extends the existing approval-bridge layer with the workspace-access prompt, wires the policy into the seams, and emits audit events through the event store.
- **`internal/api/*` (modified, phase 0 only)** — the session owner projection (`GET /api/sessions/:session_id/owner`) for the web deep-link confirm.
- **`web/` (modified, phase 0 only)** — deep-link confirm on the owner projection (ADR-004).

Data flow (agent path): tool call / HTTP request / claim / spawn → seam detects workspace mismatch → build `workspaceaccess.Request{Actor, Target, Seam}` from daemon-validated identity → `Policy.Authorize` → decision (+ one audit event) → allow proceeds with unchanged downstream behavior; deny keeps today's error shape (404-masking preserved) plus the mode hint. On the native-tool path with a live session, a `PromptEligible` deny (mode `approve-reads`, no session consent yet) goes through the prompt at the same dispatch layer tool approvals use today; the answer applies once or for the session's lifetime.

### Architectural Boundaries

- `internal/workspaceaccess` imports stdlib + `internal/logger` only; the session mode arrives via the injected `ModeSource`, consent via the injected `SessionConsentCache`, events via the injected `AuditEmitter`. It MUST NOT import `internal/config`, `internal/events`, `internal/store/*`, `internal/session`, `agentidentity`, `task`, `tools`, `workspace`, `daemon`, or `api/*`.
- Consumers import `workspaceaccess` downward and construct `ActorRef` from their own validated identity (identity snapshot, task actor context, native bound-session lookup). Interfaces defined where consumed; no back-pointers.
- `internal/daemon` is the sole composition root (mode source, consent cache, policy, bridge extension).
- `magefiles/boundaries.go` gains the `workspaceaccess` rules in the same commit that creates the package.

## Implementation Design

### Core Interfaces

```go
// internal/workspaceaccess/policy.go
package workspaceaccess

type ActorKind string

const (
    ActorAgentSession ActorKind = "agent_session"
    ActorHuman        ActorKind = "human"
    ActorExtension    ActorKind = "extension"
    ActorAutomation   ActorKind = "automation"
    ActorNetworkPeer  ActorKind = "network_peer"
    ActorDaemon       ActorKind = "daemon"
)

// ActorRef fields derive from daemon-validated identity (identity snapshot,
// task actor context, or the native bound-session lookup) — never from
// caller-supplied headers or input bodies. AgentName is best-effort audit
// attribution only; no decision ever depends on it.
type ActorRef struct {
    Kind        ActorKind
    SessionID   string
    WorkspaceID string // canonical workspace id from the registry; "" for workspace-less sessions
    AgentName   string
    Operator    bool
}

type Seam string

const (
    SeamIdentity     Seam = "identity"
    SeamTask         Seam = "task"
    SeamTool         Seam = "tool"
    SeamSpawn        Seam = "spawn"
    SeamCoordination Seam = "coordination"
)

type Request struct {
    Actor             ActorRef
    TargetWorkspaceID string // canonical workspace id from the registry; validated at step zero
    Seam              Seam
}

type DecisionSource string

const (
    SourceOperator       DecisionSource = "operator"
    SourceSameWorkspace  DecisionSource = "same_workspace"
    SourcePermissionMode DecisionSource = "permission_mode"
    SourceSessionConsent DecisionSource = "session_consent"
    SourceDenied         DecisionSource = "denied"
)

type Decision struct {
    Allowed bool
    Source  DecisionSource
    // PromptEligible is true only on approve-reads denials with no session
    // consent recorded. Only the native-tool seam acts on it; every other
    // seam treats the decision as a plain deny.
    PromptEligible bool
}

// Policy fails closed on every internal error and validates the request
// before any policy branch. One audit record per call, best-effort append.
type Policy interface {
    Authorize(ctx context.Context, req Request) (Decision, error)
}
```

```go
// internal/workspaceaccess/default_policy.go

// Mode mirrors config.PermissionMode values without importing config.
type Mode string

const (
    ModeDenyAll      Mode = "deny-all"
    ModeApproveReads Mode = "approve-reads"
    ModeApproveAll   Mode = "approve-all"
)

// ModeSource resolves the effective permission mode of a live session.
// Unknown session, empty mode, or a read failure is an error and denies.
type ModeSource interface {
    SessionPermissionMode(ctx context.Context, sessionID string) (Mode, error)
}

// Consent is a session-lifetime answer captured from the ACP prompt.
type Consent string

const (
    ConsentAllow  Consent = "allow"
    ConsentReject Consent = "reject"
)

// SessionConsentCache is daemon-owned, in-memory, keyed by session ID, and
// cleared when the session stops. Lookup misses return ok=false.
type SessionConsentCache interface {
    ConsentFor(ctx context.Context, sessionID string) (Consent, bool)
    PutConsent(ctx context.Context, sessionID string, consent Consent)
}

// AuditEmitter appends one AccessRecord per decision. Best-effort: an append
// error is logged Warn and never changes the decision.
type AuditEmitter interface {
    EmitWorkspaceAccess(ctx context.Context, record AccessRecord) error
}

type AccessRecord struct {
    Actor    ActorRef
    TargetID string
    Allowed  bool
    Source   DecisionSource
    Mode     Mode
    Seam     Seam
    Err      string
}

// Deps are all required; New errors on any nil (cheap structural fail-closed).
type Deps struct {
    Modes   ModeSource
    Consent SessionConsentCache
    Audit   AuditEmitter
    Log     *slog.Logger
}

func New(deps Deps) (*DefaultPolicy, error)
```

Decision order (fixed; owned exclusively by the unit suite): **(0)** structural validation — unknown kind, empty/non-canonical target, empty session ID on an agent-session actor → error deny before any branch; **(1)** operator → allow; **(2)** `Kind != ActorAgentSession` → deny (the mode never widens extension/automation/network-peer/daemon actors); **(3)** same-workspace → allow; **(4)** mode lookup — error or unrecognized value → error deny; `approve-all` → allow (`Source: permission_mode`); `deny-all` → deny; **(5)** `approve-reads` → consent cache: `allow` → allow (`Source: session_consent`); `reject` → deny; miss → deny with `PromptEligible: true`.

**Prompt wiring:** the workspace-access prompt hangs on the same daemon approval-bridge layer tool approvals use today (`RequestToolApproval` pattern, `internal/daemon/tool_approval_bridge.go` as template): when the native-tool path receives a `PromptEligible` deny and the session is live, the bridge issues `session/request_permission` with a daemon-computed option set — `allow_once | allow_session | reject_once | reject_session` (ACP kinds `AllowOnce`/`AllowAlways`/`RejectOnce`/`RejectAlways`; the "always" options are named and scoped *for this session*) — approval-timeout bounded (default 120s; timeout denies). `allow_once`/`reject_once` apply to that call only; `*_session` answers are stored in the consent cache and govern **all seams** for the rest of the session. Timeout, transport failure, or out-of-set answers deny and store nothing. The prompt detaches from the originating request context (`context.WithoutCancel` + bridge deadline, L-001).

**Mode provenance:** `ModeSource` is implemented in the daemon against the session manager's live session state (the same `EffectivePermissions` the manager loads at start, `internal/session/manager_start.go`); sessions the manager cannot resolve deny. The mode is start-resolved and immutable for the session's lifetime, so no live-reload semantics apply.

**Actor provenance:** each seam builds `ActorRef` from what it already validates — `agentidentity` from the session snapshot; `task` from `ActorContext` (no shape change: `AgentName` stays best-effort empty where the seam does not hold it); the native seam from the existing bound-session lookup (`nativeBoundSession`), with `Kind` fixed by the construction site, never inferred from field presence.

Error conventions: seams keep their existing sentinels (`ErrIdentityUnauthorized`, `task.ErrPermissionDenied`, 404-masked variants); the tool seam adds `ReasonWorkspaceAccessDenied` for policy denials (`ReasonScopeMismatch` stays for identity-field conflicts); denial copy gains the hint `cross-workspace access is denied by this session's permission mode; ask the operator to set the agent's permissions.mode to approve-all, or approve the prompt when asked`.

### Data Models

**None.** No new table, no schema fragment, no migration, no config key. The decision inputs are the session's existing `EffectivePermissions` (persisted in `SessionMeta`, loaded by the manager) and a daemon-owned in-memory consent map. The audit trail reuses the existing `EventSummary` store (append-only observability, not matchable state).

`permissions.mode` (agent definitions) and `[permissions] mode` (global default) already exist, are already operator-managed (config file, agent definition surfaces, Settings), and are already protected from agent mutation: `permissions.*` is a config trust root — `ClassifyToolConfigPath` returns `ConfigPathTrustForbidden` for `compozy__config_set` (`internal/config/tool_surface.go`, `tool_surface_security.go`). This spec changes none of that.

### API Endpoints

Phase 0 only (ADR-004). Handler lands in `internal/api/core` (`session_owner.go`); HTTP/UDS register with the existing guards. Contract type in `internal/api/contract/`; OpenAPI + TS via `make codegen` (skill `eng-contract-codegen-coship`).

| Method | Path | Description |
| --- | --- | --- |
| `GET` | `/api/sessions/:session_id/owner` | Owner projection for deep links: exactly `{session_id, workspace_id, workspace_name}`; 404 unknown. Operator-scope global read (like `GET /api/sessions`), minimal by design |

No workspace-access status/grant routes, no CLI verbs, no native tools, no Settings section — deleted from the design by ADR-007.

## Safety Invariants

1. **Explicit non-permissive outcomes preserved, deltas named.** With an explicit `approve-reads` mode and no prompt answer, every cross-workspace agent request outside the native-tool seam is denied with today's authorization outcome, status, and sentinel. Named public deltas: (a) `approve-all` sessions cross freely — the deliberate product change and the built-in configuration default; (b) the native-tool seam prompts in `approve-reads` instead of hard-denying; (c) tool-seam reason `ReasonScopeMismatch` → `ReasonWorkspaceAccessDenied` for workspace-axis denials; (d) denial copy gains the mode hint; (e) the CLI-local pre-flight denial becomes daemon-origin (same exit 77). Existing isolation suites pass with wiring-only edits plus these deltas.
2. **Fail closed in the decision core.** Step-zero request validation; required constructor deps; mode lookup errors and unrecognized modes deny; unknown sessions deny.
3. **Canonical IDs only.** `Authorize` receives only workspace ids the registry returns after reference resolution (`Workspaces.Resolve`). Both persisted canonical forms are accepted: registration ids (`ws_` + 16 lowercase hex) and durable 26-character Crockford ULIDs. Raw names and paths never reach the policy.
4. **Actor-kind gate.** Only validated `agent_session` actors are eligible for mode-based allowance; other kinds keep current behavior at every seam; `ActorRef` derives from daemon-validated identity, with `Kind` fixed by the construction site.
5. **Operator semantics unchanged.** `Operator == true` bypasses exactly where operator bypasses exist today; no new operator entry points.
6. **No agent self-widening on guarded surfaces.** `permissions.*` is already trust-root-forbidden for `compozy__config_set`; this spec adds no mutable state an agent could reach. Direct config-file edits remain the pre-existing, enumerated ADR-006 gap (audit-visible, documented).
7. **`deny-all` never prompts.** On the workspace axis it is a hard deny at every seam, including the tool seam.
8. **Prompt containment.** The ACP prompt fires only from the native-tool dispatch layer on a `PromptEligible` deny with a live session, timeout-bounded, with a daemon-computed option set; timeout, transport failure, or out-of-set answers deny and persist nothing.
9. **Consent is session-scoped and volatile.** `*_session` answers live in daemon memory keyed by session ID, apply to all seams, die on session stop/daemon restart, and are visible only through the audit trail. No durable consent exists.
10. **Audit every decision.** One `AccessRecord` per `Authorize` call (spawn's two validation phases = two records, documented), appended best-effort before callers convert denials into 404-masked shapes; append failures log `Warn` and never change the decision. `EventSummary.WorkspaceID` = actor home workspace (global when workspace-less); target/seam/source/mode in the payload.
11. **Identity axes stay hard.** `session_id`/`agent_name`/`actor_kind` conflicts remain unconditional `ReasonScopeMismatch` denials; the policy governs only the workspace axis.
12. **Integrity checks stay.** Window-manager snapshot/topology assertions never consult the policy.
13. **Spawn double-validation preserved.** Both `prepareSpawn` validations run and audit; a spawn that crosses workspaces is judged by the *parent* session's mode.

## Enforcement Seam Changes

| Seam | File(s) | Change |
| --- | --- | --- |
| Agent identity | `internal/agentidentity/identity.go` | `ValidateWorkspaceAccess` becomes policy-aware (ctx + `Policy` + `Request` from the validated snapshot); `Resolve` consults `opts.WorkspaceAccess`; nil policy = deny on mismatch. Copy gains the hint. |
| HTTP agent routes | `internal/api/core/agent_identity.go` | `resolveAgentCallerForWorkspace` threads the daemon-wired policy. |
| CLI pre-flight | `internal/cli/workspace_resolution.go` | Local cross-workspace hard-block deleted; daemon decides; provenance notes `cross_workspace_attempt`. |
| Task domain | `internal/task/*` | Mismatch for non-operator actors → build `ActorRef` from `ActorContext` (no `CallerScope` shape change), consult policy (via `WithWorkspaceAccessPolicy` option, pattern `WithWorkAdmissionChecker`). 404-masking preserved. |
| Tool dispatch + native binder | `internal/tools/dispatch.go`, `internal/daemon/native_workspace_input_binder.go`, `native_tool_scope.go` | Workspace-mismatch branches consult policy (actor from the bound-session lookup). `PromptEligible` deny with a live session → prompt via the approval-bridge layer; final deny returns `ReasonWorkspaceAccessDenied`. Envelope `workspace_id` conflicts consult the same policy without a prompt (the check stays at its current dispatch position; only its deny branch changes). |
| Spawn | `internal/session/spawn.go` | Cross-workspace spawn consults policy with the parent's session (no prompt); both phases audit; allowed spawn binds the child to the foreign workspace normally. |
| Coordination | `internal/workspace/coordination_commands.go` | Workspace compare consults policy via the caller scope; writes still require operator. |
| NOT seams | `workspace_scope.go` (`requireSessionInWorkspace`), `windowmanager/*`, `network/participation` | Unchanged: URL consistency, store integrity, data-model consistency. |

## Web: Operator Deep-Link Confirm (Phase 0)

Per ADR-004, unchanged by the re-anchoring: when the session deep-link loaders (`web/src/routes/_app/-agent-session-route-loader.ts`, `-session-permalink-route.ts`) miss in the active workspace, they call `GET /api/sessions/:session_id/owner` — which returns exactly `{session_id, workspace_id, workspace_name}` — cached under a dedicated `session-owner` query key, never merged into session catalogs or detail caches (no foreign payload client-side before confirmation). A routed confirmation dialog offers "switch to workspace B?"; confirm activates the owning workspace and re-enters the route; cancel keeps today's not-found rendering; a 404 keeps today's behavior with no dialog. Operator UX only — agents are governed by the policy, never web routing. Lands on top of the in-flight `xstate-refac` loader refactor.

The implementation's two-touch corrective design is normative in
`corrections/task-04-route-ownership/_techspec.md` and ADR-008: `/agents/$name` is a search-neutral
parent layout, its index leaf owns agent-detail search/load/sync, and the session leaf exclusively
owns `workspaceSwitch`.

## Delete Targets

Code deletions (hard cuts in the same change as their replacement):

- `internal/agentidentity/identity.go` — pure two-string `ValidateWorkspaceAccess`; all callers updated same change.
- `internal/cli/workspace_resolution.go` — `validateResolutionAgainstSessionIdentity` + call sites (daemon is the sole authority; behavioral coverage in the canonical resolution suite, no tombstone tests).
- `internal/daemon/native_workspace_input_binder.go` / `native_tool_scope.go` — inline workspace-mismatch deny branches (workspace axis only; identity-field branches stay).
- Identity denial copy `"use the session workspace or start a session in the requested workspace"` — replaced by the hint copy.
- `web/` — the unconditional foreign-session not-found branch in the two deep-link loaders.
- `docs/qa/scenarios/ET-web-session-deep-link-isolation.md` — expected text rewritten; `qa_status` → `untested`.

Design-record deletions (spec-level, nothing implemented — the ADR-002/003 program removed by ADR-007):

- `workspace_access_grants` table + sqlc queries + repo; `[workspace_access]` config section + lifecycle/bool config-set work; `GET/PUT/DELETE /api/workspace-access*` routes; `compozy workspace access` CLI verbs; `compozy__workspace_access` native toolset; web Settings "Workspace access" section; grant transition events (`workspace.access_grant_put/revoked`); the dedicated standing system-prompt rule and its presence tests; the `CallerScope.AgentName` hard cut. ADR-005's machinery stays unbuilt (superseded design record).
- No schema deletions; no existing migration bytes change.

## Accepted Beta Gaps (ADR-006, re-scoped by ADR-007)

Enumerated for docs and QA. Audit coverage is not uniform across these gaps and is stated per gap below:

1. An agent-driven shell can edit config files on disk and raise a `permissions.mode` (its own agent's definition or the global default). Pre-existing product gap for the whole `permissions.*` axis, not new to this feature; the hard fix is the named future containment ADR.
2. Session consent has no management surface: the operator cannot list or revoke a live session's `allow_session` answer short of stopping the session. Bounded by session lifetime; every consented decision is audited.
3. The owner projection and global session reads remain loopback-open operator-surface reads (same posture as `GET /api/sessions` today).
4. **Operator boundary is not technically enforced.** Local HTTP and UDS requests that arrive without agent identity headers are trusted as the operator — the single-operator beta assumption. A deliberately adversarial managed session can therefore remove its injected identity environment or headers and call those operator surfaces directly, including workspace detail and Compozy Network routes, reaching data its seam checks would deny. This spec's seams constrain identified sessions; they do not authenticate the operator. Audit: headerless calls are not session-attributable at the workspace-policy boundary — the Network middleware skips the workspace-access policy check and its `workspace.access_*` audit when identity headers are absent, so the underlying operations may carry their own audits but no workspace-access audit is guaranteed. The gap is detection and instructional enforcement only until the future OS containment or technical operator-authority ADR closes it — nothing here prevents it.

## Extensibility Integration Plan

- **Extensions / Host API**: unaffected — bound-workspace binding map and ceilings unchanged; invariant 4 keeps the mode from widening extension actors. Checked: `hostAPIWorkspaceBindings`, `manager_supervision` ceilings, task-seam actor kinds.
- **Hooks**: no new hook events; claim/spawn hook semantics unchanged (hooks narrow, never widen). Checked: hooks taxonomy, `lease_claim_hooks.go`, `spawn_hooks.go`.
- **Skills/capabilities, bundles, registries, bridge SDKs, MCP sidecars**: no contract change; `compozy mcp serve` binding untouched. The bundled `skills/compozy/` skill gains the mode-mapping documentation (below). Checked: serve binding, bridges routing, bundle activation.
- **Native tools/resources**: no new tools. The binder deny branch changes reason code to `ReasonWorkspaceAccessDenied` and gains the prompt path; no tool schemas change.
- **Protocol docs**: OpenAPI + TS regenerate for the owner projection only; no Compozy Network wire change.

## Agent Manageability Plan

No new manageable state exists, so no new surfaces are built — the mode is the knob and it is already managed:

- **Discover**: denials are deterministic and machine-parseable (`identity_unauthorized` + hint; `ReasonWorkspaceAccessDenied`; exit 77). The hint names `permissions.mode` and the prompt path.
- **Operate (operator)**: set the agent's `permissions.mode` (agent definition surfaces / config file / existing Settings) or the global `[permissions] mode` default; answer prompts in `approve-reads`.
- **Operate (agent)**: hitting the boundary in `approve-reads` triggers the prompt; in `deny-all` the denial is final and the agent should surface it to the operator. `compozy__config_set` on `permissions.*` fail-closes (`ConfigPathTrustForbidden`, existing behavior).
- **Audit**: `EventSummary` surfaces (`GET /api/logs`, `compozy logs --type <event-type>`,
  `compozy__logs`, and `compozy__observe_search`) filterable by `Type = workspace.access_*`,
  `ActorKind`, and actor-home `WorkspaceID`.

## Config Lifecycle

**No config change.** No new keys, no overlay/scope/lifecycle/mutability edits. `[permissions] mode` and per-agent `permissions.mode` already exist with their current semantics; `permissions.*` is already trust-root-protected from `compozy__config_set`. Docs gain the cross-workspace mapping of the existing modes (below).

## Web/Docs Impact

- Phase 0: two deep-link loaders + `active-workspace-store` + `session-owner` query key module (coordinates with `xstate-refac`).
- No Settings work (deleted by ADR-007).
- `packages/site`: security/isolation page documents the mode mapping (`approve-all` crosses, `deny-all` never, `approve-reads` asks), the session-scoped consent semantics, the `deny-all` per-axis asymmetry, and the beta posture + accepted gaps in plain language; permissions reference page gains the cross-workspace column.
- `skills/compozy/`: mode mapping, prompt semantics (`allow_once|allow_session|reject_once|reject_session`), deny-hint contract, audit event types.
- `docs/_memory/glossary.md`: "Cross-workspace access" entry (mode-anchored; replaces the planned "Workspace access grant" entry).
- **QA tracker**: reset `ET-web-session-deep-link-isolation` (rewritten expectations); new content-addressed `untested` scenarios: mode matrix across seams (approve-all / deny-all / approve-reads), prompt outcome matrix incl. session consent reuse and expiry-on-stop, deep-link confirm. Flag, don't retest.

## Impact Analysis

| Component | Impact Type | Description and Risk | Required Action |
| --- | --- | --- | --- |
| `internal/workspaceaccess` | new | Decision core; low risk | Package + boundaries rule |
| `internal/agentidentity` | modified | Policy-aware validate; medium risk | Signature + copy |
| `internal/task` | modified | Policy consults at 3 sites (no type shape change); medium risk | Option + consults |
| `internal/tools` + `internal/daemon` | modified | Policy consult + prompt on existing bridge layer + mode source + consent cache; medium risk | Binder/dispatch deny-branch changes, bridge extension, cache + session-stop cleanup |
| `internal/session` spawn / `internal/workspace` coordination | modified | Policy consults; low-medium risk | Wire + audit |
| `internal/api/*` | new surface (phase 0) | Owner projection; low risk | Handler + routes + codegen |
| `internal/cli` | modified | Pre-flight block deleted; low risk | Deletion + provenance note |
| `web/` | modified | Deep-link confirm; medium risk (refactor overlap) | Loader changes + e2e |
| `packages/site`, `skills/compozy/`, `docs/qa/` | modified | Docs/QA truth incl. mode mapping + beta posture | Same-change updates |

## Testing Approach

Strategy only; cases in `_tests.md`. Suite ownership is enforced: the policy decision chain lives **only** in the `workspaceaccess` unit suite; integration cases each prove one real substrate boundary (live session-mode lookup, consent cache lifecycle through the daemon, audit append, owner-projection route wiring) with the minimum representative decision needed; E2E proves cross-surface journeys (prompt outcomes, mode matrix through real seams, deep-link confirm) without re-asserting precedence. Regression floor: existing isolation suites pass with wiring-only edits plus the named deltas of invariant 1. Conventions: `t.Run("Should …")` + `t.Parallel`, `-race`, fakes at I/O boundaries only; frontend via `bunx turbo run lint typecheck test --filter=./web`; completion gate one full `make verify`.

## Development Sequencing

### Build Order

1. `internal/workspaceaccess` (validation, chain, deps, audit contract).
2. Daemon `ModeSource` (session manager lookup) + in-memory consent cache with session-stop cleanup.
3. Events registry entries + `EventSummary` audit emitter.
4. Seam wiring (identity → task → tools/binder → spawn → coordination) + delete-target sweep.
5. Daemon composition: policy + prompt on the approval-bridge layer.
6. API contract + owner projection handler + routes + codegen (phase 0 backend — parallelizable from step 1).
7. Web phase 0 (owner projection consume + confirm flow).
8. Docs (site, official skill, glossary) + QA scenario flags.

### Technical Dependencies

- None external. Internal: `xstate-refac` loader overlap (phase 0 lands on top).

## Monitoring and Observability

- Events: `workspace.access_denied` → `notify(global(warning))`; `workspace.access_granted` → `global(success)`. No grant/toggle event types exist. All written with `context.WithoutCancel`; append failures log `Warn`.
- Record scoping: `WorkspaceID` = actor home (global when workspace-less); target/seam/source/mode/error in payload; `SessionID`/`AgentName`/`ActorKind` in summary columns.
- Closes today's gap where an `agentidentity` 403 is fully silent.
- Metrics: out of scope v1.

## Technical Considerations

### Key Decisions

ADR-001 (central policy at seams, amended: decision source is the session mode), ADR-004 (deep-link confirm + minimal owner projection), ADR-006 (beta posture, amended: dedicated rule artifact removed), ADR-007 (PermissionMode anchoring; supersedes ADR-002/003). ADR-005 remains the superseded future-hardening design record.

### Known Risks

- **Coupled axes** — `approve-all` opens both tool risk and the workspace boundary; the orthogonal configuration is gone. Named product decision (ADR-007); revisit only with evidence, as a new TechSpec.
- **Spawn-mediated mode escalation** — an `approve-reads` parent spawns a same-workspace child whose agent definition is `approve-all`; the child crosses on its own authority. Exact parity with the existing tool-risk axis; agent definitions are operator-authored; accepted (ADR-007 risk section).
- **Seam drift** — future deny sites bypassing the policy; mitigations: seam list in package docs, parity journeys, candidate lesson after landing.
- **`xstate-refac` overlap** — phase 0 lands on top of the refactor.

## Compozy Impact Audit

- **Native tools**: no new tools or schema changes; binder workspace-mismatch deny branch → policy consult + `ReasonWorkspaceAccessDenied` + prompt path. Checked: toolmeta registry, descriptors, capability gates — unchanged.
- **Extensibility and hooks**: Host API ceiling unchanged, protected by the actor-kind gate; no new hook events; MCP serve binding unchanged; bundles/bridges/registries unchanged (checked routing/binding/activation sites); no config lifecycle change.
- **Workspace data isolation**: non-permissive-mode defaults preserved at every seam (invariant 1); opt-in crossings audited; the owner projection exposes only three fields; consent is session-scoped memory that cannot outlive its session; accepted beta gaps enumerated in-spec and documented on the site.
- **Official Compozy skill**: `skills/compozy/` updated with the mode mapping, prompt semantics, deny-hint contract, and audit event types (same change).

## Architecture Decision Records

- [ADR-001: Central policy at existing enforcement seams](adrs/adr-001.md) — amended by ADR-007 (decision source).
- [ADR-002: All-or-nothing access model](adrs/adr-002.md) — **superseded by ADR-007**.
- [ADR-003: Grants clone the tool-approval template](adrs/adr-003.md) — **superseded by ADR-007** (prompt survives at the tool seam; durable grants removed).
- [ADR-004: Deep links prompt before switching workspaces](adrs/adr-004.md) — unchanged.
- [ADR-005: Operator-presence boundary](adrs/adr-005.md) — superseded; design record for future hardening.
- [ADR-006: Beta posture — instructional enforcement](adrs/adr-006.md) — amended by ADR-007 (posture stands; rule artifact removed).
- [ADR-007: Cross-workspace access anchored on the session PermissionMode](adrs/adr-007.md).
- [ADR-008: Agent detail is an index leaf under a search-neutral parent layout](adrs/adr-008.md).

## Assumptions and Defaults

- Single-operator local daemon; beta threat model per ADR-006: supervised, instruction-following agents are constrained technically at the seams; adversarial agents are detected (audit), not prevented — gaps enumerated above.
- Fresh/upgraded installs retain the built-in `approve-all` configuration default. Tests and operators
  select `approve-reads` explicitly when exercising prompt/deny behavior; the named deltas of invariant
  1 are the only behavior changes for that explicit mode.
- An allowed cross-workspace actor gets the same rights in the target workspace it has at home — no read-only tier.
- Workspace-less sessions follow the same mode chain (same-workspace never matches; consent is keyed by session ID, so session-scoped answers work for them too).
- Dream sessions are pinned `approve-all` by the session manager today; they are daemon-driven, not agent-request-driven, and inherit the mode mapping as-is.
- "No scope requested" (empty target) is resolved by callers before `Authorize`; the policy rejects empty targets at step zero.
- Memory `global` scope, operator global listings, `GET /api/workspaces`, and the owner projection remain intentionally global operator-surface reads.
- No compat shims, no fallback paths, no placeholder flags: delete targets are removed in the same change that lands their replacement.
