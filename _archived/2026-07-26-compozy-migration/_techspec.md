# TechSpec — Compozy v0.3.0 Migration

## Executive Summary

This design hard-cuts AGH into Compozy v0.3.0 and closes the approved P0
parity gaps: per-task runtime selection with durable observability, review
artifacts, bundled development skills, per-loop input defaults, explicit
configuration migration, mirrored migration guidance, and run deep links.
Existing packages and service boundaries remain authoritative; the migration
does not introduce a compatibility layer, a generic artifact service, or a
second runtime configuration system.

No `_prd.md` or `_user_stories.md` exists for this program. The approved
`_brief.md` is the product input, ADR-001 through ADR-007 record the settled
design decisions, and `_tests.md` is the concrete verification contract.
Release publication, repository administration, DNS, hosting, and channel
promotion remain human-run operations after pre-publish QA.

## System Architecture

### Component Overview

| Component | Responsibility |
| --- | --- |
| `internal/loop` | Resolve runtime and effective inputs, separate static from effective validation, and expose truthful run state. |
| `internal/daemon` | Apply resolved runtime through the existing provider policy gate and supply trusted workspace authority to extensions. |
| `internal/config` | Decode and merge the v0.3 configuration shape while preserving value presence and origin. |
| `internal/store/globaldb` | Persist resolved runtime in the existing workspace-owned loop output model. |
| API contracts, HTTP, UDS, native tools, CLI | Share generated DTOs and service behavior; no surface owns a private projection. |
| `extensions/dev-cycle` | Own review artifacts, task-frontmatter import, loop graphs, and bundled development resources. |
| Web and documentation site | Render read-only runtime truth and publish the migration/launch contract without inventing controls or release state. |
| Release workflow | Pin `github.com/compozy/releasepr@v0.0.24`, consume its explicit release-plan outputs, and retain ownership of tag creation and publication. |

The loop engine resolves execution intent. The daemon remains the only layer
that derives provider command, authentication, home, environment, and
credential policy. Extension code receives trusted workspace identity through
the host boundary and never imports daemon internals or accepts a caller-chosen
filesystem root.

The implementation extends existing packages and generated-contract flows.
It does not create a generic event bus, parallel run-status store, legacy
configuration reader, or new release service.

## Implementation Design

### Core Interfaces

Runtime configuration uses one shape across loop definitions, task
frontmatter, rules, node parameters, overrides, binding, and public status:

```go
type RuntimeSpec struct {
	Provider  string
	Model     string
	Reasoning string
}

type ResolvedRuntime struct {
	Runtime RuntimeSpec
	Source  RuntimeProvenance
}

type RuntimeResolver interface {
	ResolveItem(context.Context, RuntimeLayers, ItemRuntime) (ResolvedRuntime, error)
}

type RuntimeCatalog interface {
	ValidateRuntime(context.Context, RuntimeSpec) error
}
```

The engine-to-daemon boundary carries the resolved intent and returns the
runtime actually applied by provider policy:

```go
type RuntimeOverrides struct {
	Provider, Model, Reasoning string
}

type ActionSessionBindRequest struct {
	Runtime RuntimeSpec
}

type ActionSessionBinding struct {
	AppliedRuntime RuntimeSpec
}

func (*Config) ResolveSessionAgentWithRuntime(
	AgentDef, RuntimeOverrides,
) (ResolvedAgent, error)
```

Runtime layers merge per field in this order:

1. invocation rules;
2. task-frontmatter `runtime`;
3. merged configuration rules;
4. rendered node `params.runtime`;
5. effective worker defaults;
6. the agent definition applied by the daemon.

Rules match exactly one of `id`, `type`, or `complexity`; specificity is
`id > type > complexity`, with later rules winning only at equal
specificity. Judges use judge defaults plus criterion runtime and never task
rules. Child loops resolve their own layers.

Static `loop validate` checks definition-contained structure. Dry-run and
submission resolve workspace/run values, inputs, frontmatter, templates,
aliases, catalogs, and provenance before any ACP process starts. Provider
identity always validates; model membership validates only when the provider
catalog is authoritative. The daemon publishes the final applied runtime, not
an inferred pre-bind value.

Review artifacts remain a dev-cycle filesystem contract. The dev-cycle
`reviewer` agent authors each round's issues through its `output_schema`
(ADR-008); that source-agnostic issue request enters a writer that
exclusively owns round allocation, serialization, and finalization. The host
supplies trusted workspace scope; the request carries a logical task name,
never a root path. The writer keeps a deterministic byte contract against
v0.3-owned goldens and rejects traversal, symlink, and concurrent-round
escape paths.

### Data Models

| Model | Contract |
| --- | --- |
| Resolved runtime | Durable per generation output/item with independent provider, model, and reasoning provenance; persisted atomically only after successful binding. |
| Per-loop input defaults | Presence-aware values under `[loops.inputs.<loop-name>]` at global and workspace scope; resolution is run > workspace > global > definition. |
| Review artifacts | `.compozy/tasks/<task>/reviews-NNN/issue_NNN.md`; deterministic bytes and monotonic `pending -> valid|invalid -> resolved` lifecycle. |
| Migration receipt | One `{key, action, targets?, reason?, guide_ref?, redacted_value?}` entry for every populated legacy leaf. |
| Bundled skills | Nine immutable first-party dev-cycle resources projected globally into eligible managed sessions without mutable workspace ownership. |

`resolved_runtime` extends the existing loop-output persistence model. It
survives store reopen and remains workspace-scoped across status, SSE, CLI,
HTTP/UDS, native tools, and web reads. It does not use a JSON fallback or a
parallel run table.

The config migrator reads the pinned v0.2.15 Go schema, targets either global
configuration or one explicit `--workspace <root>`, and never infers scope
from the current directory. Every decoded legacy leaf is translated or
dropped exactly once, explicit false/zero values remain present, and
unsupported values are reported rather than weakened. Task 09 owns the single
canonical leaf matrix used by its tests and both migration-guide surfaces.

Before mutation, the migrator validates the legacy shape and writes a
timestamped backup/marker. A second run for the same target refuses. Runtime
state is not migrated or deleted. Pre-decode diagnostics report legacy format
and colliding legacy-owned `agents/` or `extensions/` directories without
loading, enrolling, rewriting, or deleting them.

### API Endpoints and Public Surfaces

| Method and path | Request / response | Contract |
| --- | --- | --- |
| `POST /api/workspaces/:workspace_id/loops/:name/run` | `RunLoopRequest` / `RunLoopResponse` | Starts a persisted run or returns a dry-run plan; effective runtime/input failures use structured 422 errors. |
| `POST /api/workspaces/:workspace_id/loops/:name/validate` | `ValidateLoopRequest` / `LoopValidationResponse` | Validates definition-contained structure without claiming workspace/run overlays. |
| `GET /api/workspaces/:workspace_id/loop-runs/:run_id` | none / `LoopRunResponse` | Returns workspace-scoped durable run, generation, and resolved-runtime detail. |
| `GET /api/workspaces/:workspace_id/loop-runs/:run_id/events` | none / SSE | Streams the same workspace-scoped persisted run truth. |

| Surface | Contract |
| --- | --- |
| CLI | `compozy loop run --workspace <ref>`, `loop validate --workspace <ref>`, `loop status --workspace <ref> --run-id <id>`, `loop runs --workspace <ref>`, and local `migrate config [--workspace <root>]`. |
| Native tools | Renamed loop run/validate/status/list and config get/set/unset tools use the same schemas and capability gates. |
| Web | The run page renders persisted runtime/provenance read-only; successful persisted runs link to `/loop-runs/<run_id>`. |
| Configuration | CLI, HTTP, UDS, and native config surfaces inspect and mutate runtime settings and dynamic loop-input defaults with structured output. |
| Documentation | Root and site migration guides carry the same eight-section command, config, state, compatibility, SDK, license, and domain contract. |

`LoopPlanPayload` owns effective dry-run inputs and per-key origin.
`RunLoopResponse` owns optional `web_url` only when a run was persisted.
Human, JSON, and TOON output derive from those DTOs. Contract changes
regenerate every owned output family and pass `make codegen-check`.

Retired fields and commands fail explicitly with migration guidance. No
unscoped loop route, `loop runs show`, `tasks validate` compatibility verb,
or dry-run run URL is introduced.

## Integration Points

| Integration | Boundary |
| --- | --- |
| ACP providers | The daemon applies the resolved triple through existing provider/session policy. Missing auth or an incompatible binding fails before spawn. |
| Model catalog | The loop layer depends on a read-only validation contract and does not claim model authority for providers without an authoritative catalog. |
| Review source | The dev-cycle `reviewer` agent authors `ReviewIssue[]` through its `output_schema` (ADR-008); v0.3.0 integrates no external review provider, PR watch, thread resolution, or `gh` surface. |
| Extension host/SDK | Trusted workspace identity and changed tool/resource schemas co-ship through manifests, descriptors, digests, and dispatch. |
| `releasepr v0.0.24` | Read-only release planning validates the explicit ref/version/channel, checked-out commit, and local/remote tag absence; its GitHub/npm/Homebrew outputs are authoritative workflow inputs. |
| Release registries, Sigstore, Homebrew, hosting, DNS | These are external operational boundaries. Tasks 10–13 prepare and verify runbooks; no implementation component performs irreversible publication or cutover. |

## Impact Analysis

| Component | Impact Type | Description and Risk | Required Action |
| --- | --- | --- | --- |
| Runtime DSL/config/binding | Modified | New runtime shape replaces scalar model fields; incorrect layering could bind the wrong provider. | Hard-delete old fields, preserve provenance, validate before bind, and test all layers. |
| Loop store/read surfaces | Modified | Runtime truth must survive reopen and remain workspace-isolated. | Append-only schema evolution and shared projections across all read surfaces. |
| Dev-cycle extension | Modified | Review artifacts and task import become public extension contracts. | Co-ship schema/digest/dispatch changes and containment/golden coverage. |
| Config lifecycle | Modified | Dynamic loop input defaults and explicit migration affect global/workspace state. | Preserve presence/origin, expose all management surfaces, and keep migration non-destructive. |
| Bundled resources | Modified | Development skills become immutable first-party extension resources. | Ship the ADR-004 set, exclude deferred resources, and prove cross-workspace isolation. |
| Web/site/docs | Modified | Runtime truth, migration guidance, branding, and release copy change. | Regenerate contracts, update the official skill/docs, and verify visible surfaces. |
| Distribution | Modified | Binary/module/channel identity hard-cuts while beta, stable, and legacy releases require one deterministic policy decision. | Pin `releasepr v0.0.24`, consume its plan without re-derivation, prepare deterministic runbooks, and defer external operations to authorized owners. |

## Testing Approach

`_tests.md` is the sole concrete case catalog. Unit coverage owns pure
resolution, parsing, mapping, validation, serialization, and idempotency.
Integration coverage uses real config/store/API/extension wiring with fakes
only at provider, filesystem-boundary, or release-I/O edges. It proves
close/reopen durability, transport parity, trusted workspace containment, and
two-workspace isolation.

End-to-end coverage follows the approved delivery, review-remediation,
in-place-upgrade, and rendered-route journeys through public CLI/API/browser
surfaces. IT-017 runs the pinned `releasepr v0.0.24` planner against the exact
candidate ref and proves commit/ref equality, unprefixed SemVer normalization,
local plus `origin` tag absence, and authoritative GitHub/npm/Homebrew outputs.
That release-PR evidence proves local intent/configuration only; live registry,
installer, signature, and channel assertions occur after an authorized publish.

Implementation uses scoped lanes while iterating and one full `make verify`
as the completion gate. The trailing QA cycle uses the repository bootstrap,
finite charter selection, strict evidence audit, browser/daemon lanes, and
mandatory clean teardown.

## Development Sequencing

### Build Order

1. Hard-cut repository, binary, module, env/home, protocol, native-tool, and
   generated-contract identity; delete old identifiers in the same changes.
2. Add runtime contracts, resolution/validation, daemon application, durable
   status, generated surfaces, and workspace-isolation coverage.
3. Add trusted review artifacts, the agent-authored `review-and-fix` loop,
   parser payload, and bundled skill authority.
4. Add input defaults, cross-surface config lifecycle, explicit migrator,
   first-boot diagnostics, guide parity, and deep links.
5. Prepare the single-cut collateral entirely in-branch — release
   identity/workflow, front door, and launch content — without publishing
   anything (brief round-11: no isolated change set, no deferred activation).
6. Run finite pre-publish QA; the authorized operator then executes the
   single-cut runbook (legacy branch, squash merge, beta publish, domain
   pointing, archival) and its post-publish live checks.

### Technical Dependencies

- External web-assets ownership and publication order must be resolved before
  Task 01 changes the module import.
- The installer serves one contract (brief round-11): Task 10 implements only
  the v0.3 Sigstore path targeting the documented beta version; no dual-asset
  transition exists and legacy installs stay with `legacy/v0.2` collateral.
- Hosting reduces to execution (brief round-11): the DNS/hosting owner points
  `compozy.com` at the site deployment during the single-cut runbook; no
  redirect-topology decision remains.
- Task 13 is pre-publish evidence. It does not publish, change DNS, or claim
  external registry/signature acceptance.

## Monitoring and Observability

The migration adds structured `runtime_validation`, `input_default`, and
`legacy.state_detected` diagnostics. Run status and SSE expose the same
persisted applied runtime/provenance as CLI, HTTP/UDS, native tools, and web.
Review tools return deterministic round/file summaries; migration emits a
machine-readable receipt.

When legacy configuration blocks daemon boot, local CLI diagnostics remain
available while HTTP/UDS truthfully do not. After migration, running-daemon
status/doctor surfaces report retained orphan paths consistently.

No new monitoring backend, metric family, alert threshold, or telemetry
service is introduced. Existing structured logs and public status/event
surfaces are sufficient for this migration.

## Technical Considerations

### Safety Invariants

1. Runtime layers merge field by field and retain independent provenance.
2. Rules have one selector; `id > type > complexity`, with later wins only
   at equal specificity.
3. Static and effective validation remain distinct; invalid execution input
   fails before ACP spawn.
4. The loop engine resolves intent; the daemon alone derives provider policy,
   and incompatible pinned bindings fail.
5. Public runtime truth comes from the applied binding and persists atomically.
6. Only trusted daemon/host scope supplies extension workspace authority.
7. Artifact paths cannot escape the trusted task tree, including under
   traversal, symlink, and concurrent-creation races.
8. The writer owns round allocation, serialization, and monotonic/idempotent
   finalization; rounds are append-only.
9. Review artifact bytes are deterministic against v0.3-owned golden
   fixtures.
10. Public schema changes co-ship generated contracts, descriptors/digests,
    matchers, docs, and official-skill guidance.
11. Migration probes and backs up before mutation, reports each key once, and
    never deletes legacy state.
12. Bundled skills are immutable global first-party resources with no mutable
    workspace authority.
13. Every workspace-owned read/write/cache/event path carries trusted
    workspace scope and rejects cross-workspace access.
14. Dynamic input-default paths are agent-manageable across CLI, HTTP, UDS,
    and native tools while preserving explicit zero values and origin.
15. Release ref, version, tag, channel, and publication policy come from one
    pinned `pr-release plan` invocation; downstream steps do not infer or
    recompute them, and the planner never creates the tag it validates.

### Key Decisions and Trade-offs

- A single field-merged runtime shape adds deterministic selection at the cost
  of a hard contract break for scalar model fields.
- Durable applied runtime favors truthful reopen/status behavior over an
  ephemeral-only projection.
- Review artifacts remain extension-owned files because no second consumer
  justifies a generic core artifact subsystem.
- Agent-authored review (ADR-008) trades external review coverage for a
  dependency-free P0 journey; external review providers have no v0.3
  surface.
- Bundled managed-session skills restore the required workflow without adding
  an external-agent home installer.
- Explicit translate-or-drop migration keeps v0.3 strict and inspectable rather
  than carrying legacy fallbacks.
- Single-cut beta delivery (brief round-11) ships every launch surface in one
  tree, while npm/Homebrew channel conventions preserve stable-channel truth;
  post-publish verification remains human-run.
- A pinned read-only release planner centralizes fail-closed release policy
  while leaving all irreversible tag and publication actions in the repository
  workflow and authorized human runbook.

### Known Risks

- The flat `.compozy` namespace can collide with legacy state; pre-decode
  probing blocks accidental enrollment and the migrator never deletes it.
- Provider model catalogs have unequal authority; validation must not reject
  unlisted models where the system lacks an authoritative catalog.
- Review artifacts touch user workspaces; trusted scope, containment, exclusive
  rounds, and byte goldens are mandatory.
- The one remaining external ownership choice (`compozy-web-assets`) can block
  its consuming stage; the ledger records it instead of guessing (brief
  round-11 resolved the hosting and installer choices).

## Delete Targets

The hard cut removes, rather than aliases:

- AGH binary/module/home/environment/native-tool/wire/network identities and
  their persisted filenames, sockets, locks, and defaults;
- `reviews-watch` — the name **and** the watch semantics — plus its old
  command/manifest/catalog references and the removed legacy review-provider
  abstraction;
- the entire CodeRabbit integration in dev-cycle (ADR-008): fetch,
  normalization, nitpick, and REST code with their types and tool
  descriptors; the PR watch source; GitHub thread resolution; the push tail;
  the `pr`/`include_nitpicks`/`poll_interval`/`quiet_period`/`auto_push`/
  `push_remote` loop inputs; the `ReviewIssue` provider provenance fields
  and provenance-based history dedupe; and the `gh` shim test dependency.
  The generic loop-engine watch-source capability (`internal/loop/watch`)
  is NOT a delete target — it stays available to user-authored loops;
- scalar runtime/model fields and all `model_defaults`/judge-model variants
  across DSL, config, binding, generated contracts, fixtures, and UI copy;
- the legacy extension minimum-version field/name;
- the retired bundled `compozy` skill and deferred
  `cy-capture-decisions` resource;
- compatibility claims for removed commands, public SDKs, and legacy state.

Tasks 01–09 enumerate the concrete files and symbols owned by each deletion.

## Architecture Decision Records

- [ADR-001: Per-Task Runtime Selection — Field-Merged `runtime`, Engine-Resolved, Binder-Applied](adrs/adr-001.md) — field-merged intent is engine-resolved and daemon-applied.
- [ADR-002: Per-Invocation Runtime Override Through Dedicated `--runtime`](adrs/adr-002.md) — runtime overrides stay outside author inputs.
- [ADR-003: `reviews-NNN/` Artifacts — Source-Agnostic Dev-Cycle Writer, File-Based Fixer](adrs/adr-003.md) — dev-cycle owns deterministic, source-agnostic filesystem artifacts.
- [ADR-004: Dev-Cycle Bundles Nine Skills From `.agents/skills`; Authoring Runs in Managed Sessions](adrs/adr-004.md) — nine immutable skills ship through managed sessions.
- [ADR-005: Distribution Identity — MIT Metadata and Active Legacy Channels](adrs/adr-005.md) — MIT metadata and active legacy channel identities remain truthful.
- [ADR-006: Flat `.compozy/` Namespace and Explicit Config Migration](adrs/adr-006.md) — no fallback, inferred workspace, or state deletion.
- [ADR-007: Per-Loop Input Defaults in Configuration (`[loops.inputs.<name>]`)](adrs/adr-007.md) — presence-aware defaults share one effective resolver.
- [ADR-008: Agent-Authored Review — `review-and-fix` Without External Providers](adrs/adr-008.md) — the reviewer agent authors each round's issues; all CodeRabbit/watch/resolve/push surface is deleted.

## AGH Impact Audit

- Native tools: renamed loop/config tools co-ship schemas, structured errors,
  descriptors/digests, capability gates, and CLI/API fallbacks; old `agh__*`
  IDs are deleted.
- Extensibility and hooks: dev-cycle changes parser output, review tools,
  trusted workspace propagation, resource manifests, loop graphs, and bundled
  skills; no new hook taxonomy, generic bundle type, or MCP sidecar is added.
- Workspace data isolation: resolved runtime, config, artifacts, status,
  SSE/cache/events, and extension calls carry trusted workspace scope.
  Bundled skills are intentionally immutable global resources, not workspace
  data.
- Official AGH skill: `skills/agh/` hard-renames to `skills/compozy/` and
  its references update with each public CLI, native-tool, config, resource,
  network, and task contract change.

## Assumptions and Defaults

The pinned legacy baseline is Compozy v0.2.15 at
`8f8908afd70c731b815e20282bacad05aa026827`; post-tag official HEAD
`c202311c8430fc0d4a7442e2dc715cabfbdc68a1` is audited separately. Current
AGH code owns destination architecture; the legacy repository supplies
behavioral evidence, not compatibility code. The approved brief and ADRs
remain binding, and the three external choices above stay unresolved until
their authorized owners decide them.
