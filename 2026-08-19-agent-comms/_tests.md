# Test Specification: Agent Comms — Typed Calls, Mailbox, and Subagents

Canonical test contract for agent-comms. Companion to `_spec.md`. Derived from
`_user_stories.md` (behavior), `_spec.md` Part II (components), `_dx.md`
(CLI/API journeys), and `_uiux.md` (browser journeys).

## Strategy

- Go: table-driven `t.Run("Should …")` subtests, `t.Parallel()` default, `-race`/`CGO_ENABLED=1`; fakes only at I/O boundaries (`calls.SessionInvoker` fake; contracts registry-store fake); real SQLite (temp dirs) in store suites; status **and** body assertions on every API case.
- Integration: `+integration` tag, co-located; store suites open real globaldb with the migration chain (fresh + reopen).
- E2E runtime: `make test-e2e-runtime` Go harness against `acpmock` — acpmock gains `call_return`/`agent_message` behaviors in the same change (runtime-contract co-ship, L-007); CLI journeys execute the exact `_dx.md` transcripts.
- E2E web: `make test-e2e-web` Playwright with the daemon fixture (`web/e2e/fixtures/test.ts`), `openAppWindow(page, "Agents", "agents")`, selectors added to `web/e2e/fixtures/selectors.ts`; no `force: true`.
- Suites extend canonical homes: session-wait/spawn invariants extend `internal/session` suites; app path-ownership extends `web/src/systems/os/lib/__tests__/app-registry.test.ts`; UDS route parity extends `internal/api/udsapi/handlers_test.go`; observability extends the coverage-matrix event test.

## Coverage Matrix

| Source | Behavior | Unit | Integration | E2E |
| --- | --- | --- | --- | --- |
| US-001 | Delegate with result contract (incl. operator caller) | UT-029..033 | IT-001, IT-002, IT-070 | E2E-001 |
| US-001.EC-1 | Unknown agent → roster error | UT-034, UT-078 | — | E2E-023 |
| US-001.EC-2 | Malformed contract rejected pre-spawn | UT-005, UT-035 | — | E2E-024 |
| US-001.EC-3 | Empty prompt rejected | UT-036 | — | — |
| US-001.EC-4 | Child-cap typed error | UT-037 | IT-003 | — |
| US-001.EC-5 | Widening atoms named | UT-038 | — | — |
| US-001.EC-6 | Restart between commit and start | — | IT-004, IT-068 | — |
| US-001.EC-7 | Two keyless calls → two children | UT-039 | IT-005 | — |
| US-002 | Follow-up same verb | UT-040 | IT-006 | E2E-002 |
| US-002.EC-1 | Expired vs never-existed | UT-041 | IT-007 | E2E-011 |
| US-002.EC-2 | Out-of-lineage denied | UT-042 | — | — |
| US-002.EC-3 | Queue behind current call | — | IT-008 | E2E-002 |
| US-002.EC-4 | Follow-up during reap | — | IT-009 | — |
| US-003 | Batch fan-out | UT-043, UT-044 | IT-010 | E2E-003 |
| US-003.EC-1 | Over-cap batch hard-reject | UT-045 | — | E2E-003 |
| US-003.EC-2 | Empty batch rejected | UT-046 | — | — |
| US-003.EC-3 | Per-item unknown agent isolated | UT-047 | IT-010 | — |
| US-003.EC-4 | Cap crossed mid-batch | UT-048 | IT-011 | — |
| US-003.EC-5 | 100× batch fast rejection | UT-049 | — | — |
| US-004 | Idempotent replay | UT-050 | IT-005 | E2E-006 |
| US-004.EC-1 | Same key different payload → conflict | UT-051 | IT-012 | — |
| US-004.EC-2 | Replay across restart | — | IT-013 | — |
| US-004.EC-3 | Concurrent same-key → one call | — | IT-005 | — |
| US-004.EC-4 | Key after retention → new call | — | IT-014 | — |
| US-004.EC-5 | Same key, different budget/deadline | UT-155 | IT-072 | — |
| US-005 | Cancel stops work (incl. deadline authority) | UT-052, UT-149, UT-155 | IT-015, IT-069, IT-071 | E2E-004, E2E-029 |
| US-005.EC-1 | Cancel terminal → typed no-op | UT-053 | — | — |
| US-005.EC-2 | Cancel vs terminal race | — | IT-016 | — |
| US-005.EC-3 | Cancel queued batch item | UT-054 | — | — |
| US-005.EC-4 | Repeated cancel idempotent | UT-053 | IT-015 | — |
| US-005.EC-5 | Child ignores stop → managed-stop | — | IT-017 | E2E-004 |
| US-005.EC-6 | Invalid deadline value rejected | UT-155 | — | — |
| US-006 | Bounded resumable await | UT-055, UT-056 | IT-018 | E2E-005 |
| US-006.EC-1 | Await unknown id | UT-057 | — | — |
| US-006.EC-2 | Timeout clamp | UT-058 | — | — |
| US-006.EC-3 | Await across restart | — | IT-019 | E2E-005 |
| US-006.EC-4 | N concurrent awaits one call | — | IT-020 | — |
| US-007 | Typed terminal return | UT-059 | IT-021 | E2E-001 |
| US-007.EC-1 | Wrapper unwrap keeps budget | UT-007 | — | — |
| US-007.EC-2 | Double return → already-settled | UT-060 | IT-022 | — |
| US-007.EC-3 | Return after cancel → superseded | — | IT-016 | — |
| US-007.EC-4 | Return vs crash race single outcome | — | IT-023 | — |
| US-007.EC-5 | Return unbound | UT-061 | — | — |
| US-008 | One repair round, errors verbatim | UT-062, UT-008 | IT-024 | E2E-007 |
| US-008.EC-1 | Contract load failure ≠ child fault | UT-009 | — | — |
| US-008.EC-2 | Repair then silence → invalid-result | — | IT-024 | E2E-007 |
| US-008.EC-3 | Error-list truncation deterministic | UT-010 | — | — |
| US-009 | Completed-without-result state | UT-063 | IT-025 | E2E-008 |
| US-009.EC-1 | Prose-only after failed extraction | UT-064 | IT-025 | — |
| US-009.EC-2 | TTL kill ≠ completed-without-result | — | IT-026 | — |
| US-010 | One budget, declared overflow (incl. per-call override snapshot) | UT-011, UT-012, UT-013 | IT-027, IT-072 | — |
| US-010.EC-1 | Budget over max rejected at pin | UT-013 | — | — |
| US-010.EC-2 | Exact boundary accepted | UT-014 | — | — |
| US-010.EC-3 | Prose result same budget | UT-065 | — | — |
| US-011 | Extraction fallback + provenance | UT-015, UT-066 | IT-028 | E2E-009 |
| US-011.EC-1 | Newest valid candidate wins | UT-016 | — | — |
| US-011.EC-2 | Extracted invalid → repair path | UT-067 | IT-028 | — |
| US-011.EC-3 | Strict mode disables extraction | UT-068 | — | — |
| US-012 | Child messages parent | UT-069 | IT-029 | E2E-010 |
| US-012.EC-1 | Lifecycle-aware queue/fail receipt | UT-070 | IT-030 | — |
| US-012.EC-2 | Over-size sender rejection | UT-071 | — | — |
| US-012.EC-3 | Restart between commit and delivery | — | IT-031 | — |
| US-012.EC-4 | Message vs terminal ordering | — | IT-032 | — |
| US-013 | Non-interrupting steer | — | IT-033 | E2E-010 |
| US-013.EC-1 | Blocked target typed refusal | UT-072 | — | — |
| US-013.EC-2 | Ordered multi-steer | — | IT-033 | — |
| US-013.EC-3 | Other-parent denied | UT-073 | — | — |
| US-014 | Receipts, closed vocabulary | UT-074 | IT-030 | — |
| US-014.EC-1 | Unknown message id | UT-075 | — | — |
| US-014.EC-2 | Terminal failed after policy | — | IT-034 | — |
| US-015 | Loop-breakers engage | UT-076, UT-077 | IT-035 | — |
| US-015.EC-1 | Ping-pong self-terminates | — | IT-035 | — |
| US-015.EC-2 | Burst ≠ loop misclassification | — | IT-036 | — |
| US-016 | Provenance + capability subtraction | UT-079 | IT-037 | E2E-012 |
| US-016.EC-1 | Commands inert | — | IT-037 | E2E-012 |
| US-016.EC-2 | No permission laundering | — | IT-037 | — |
| US-016.EC-3 | Operator-claim contradicted by stamp | UT-080 | — | — |
| US-017 | Wake carries result | UT-081 | IT-021, IT-038 | E2E-001 |
| US-017.EC-1 | Parent down → durable delivery | — | IT-031 | — |
| US-017.EC-2 | N completions, stable order | — | IT-039 | — |
| US-017.EC-3 | Completion storm: none denied | UT-082 | IT-040 | — |
| US-018 | Park + revive | UT-104, UT-105 | IT-041 | E2E-002 |
| US-018.EC-1 | Revive over budget → queued | UT-106 | IT-042 | — |
| US-018.EC-2 | Racing revivals → one | — | IT-042 | — |
| US-018.EC-3 | Definition drift surfaced | UT-107 | — | — |
| US-019 | TTL typed errors | UT-041, UT-108 | IT-007 | E2E-011 |
| US-019.EC-1 | Reap-window determinism | — | IT-009 | — |
| US-019.EC-2 | Config change forward-only | UT-109 | — | — |
| US-020 | Parent-terminal drain | UT-110 | IT-043 | E2E-013 |
| US-020.EC-1 | Completed result survives drain | — | IT-043 | — |
| US-020.EC-2 | Crash-recovery drain | — | IT-044 | — |
| US-020.EC-3 | Depth-3 subtree walk + cycle guard | UT-111 | IT-044 | — |
| US-021 | Description field | UT-112..114 | IT-045 | E2E-030 |
| US-021.EC-1 | Length bound at authoring | UT-115 | — | — |
| US-021.EC-2 | Markup inert in roster | UT-116 | — | — |
| US-022 | Injected roster | UT-117 | IT-046 | E2E-014 |
| US-022.EC-1 | Bounded large roster | UT-118 | — | — |
| US-022.EC-2 | Zero definitions state | UT-119 | — | E2E-021 |
| US-023 | Depth-3 recursion, strip + literal | UT-120, UT-121 | IT-047 | E2E-014 |
| US-023.EC-1 | Name-cycle within walls | — | IT-047 | — |
| US-023.EC-2 | Depth raise forward-only | UT-122 | — | — |
| US-023.EC-3 | Deep fan-out capped visibly | — | IT-048 | — |
| US-024 | Roster manageability | UT-088 | IT-045 | E2E-025, E2E-030 |
| US-024.EC-1 | Shadowed rows distinguishable | UT-123 | — | — |
| US-024.EC-2 | Concurrent update+list snapshot | — | IT-049 | — |
| US-025 | Loop settle contract parity | UT-017 | IT-050, IT-051 | — |
| US-025.EC-1 | Lost-keys caught at settle | — | IT-051 | — |
| US-025.EC-2 | Contract on non-payload node → lint | UT-018 | — | — |
| US-026 | Task result contract | UT-124 | IT-052 | E2E-026 |
| US-010.AC-4 | Override capped + snapshotted | UT-013 | IT-072 | — |
| US-026.EC-1 | In-flight run keeps start-time contract | — | IT-052 | — |
| US-026.EC-2 | Task overflow identical | — | IT-053 | — |
| US-027 | One regime everywhere | UT-019, UT-020 | IT-054 | — |
| US-027.EC-1 | Sentinel refs replaced by kinds | UT-021 | IT-055 | — |
| US-027.EC-2 | Redaction preserves contract | UT-022, UT-023 | — | — |
| US-028 | Agents app (tree/inbox/detail) | UT-125..127, UT-147 | — | E2E-015..017 |
| US-028.EC-1 | Empty state teaches | UT-128 | — | E2E-021 |
| US-028.EC-2 | Deep/wide tree navigable | UT-129 | — | E2E-022 |
| US-028.EC-3 | SSE resync | — | — | E2E-018 |
| US-028.EC-4 | Stale-action feedback | — | — | E2E-018 |
| US-029 | Session Calls panel + wake cause | UT-130 | — | E2E-019 |
| US-029.EC-1 | Hundreds of calls bounded | UT-131 | — | — |
| US-029.EC-2 | Pruned counterpart degrades | UT-132 | — | — |
| US-030 | Needs-you signals | UT-133 | — | E2E-020 |
| US-030.EC-1 | Auto-resolve clears | UT-134 | — | E2E-020 |
| US-030.EC-2 | Coalesced per tree | UT-135 | — | — |
| US-031 | Truthful cost | UT-136 | IT-040 | E2E-016 |
| US-031.EC-1 | Deep fan-out attribution | — | IT-040 | — |
| US-031.EC-2 | Restart never double-counts | — | IT-031, IT-040 | — |
| US-032 | Subset-only narrowing | UT-038, UT-137 | IT-056 | — |
| US-032.EC-1 | Hook widening rejected | UT-138 | IT-056 | — |
| US-032.EC-2 | Current effective set per call | UT-139 | — | — |
| US-033 | Cross-workspace hard-denied | UT-140 | IT-057 | E2E-024 |
| US-033.EC-1 | Same-name across workspaces | — | IT-057 | — |
| US-033.EC-2 | Workspace deletion terminalizes | — | IT-058 | — |
| US-034 | Secret redaction on payloads and messages | UT-022..024, UT-148 | IT-059 | — |
| US-034.EC-1 | Split-secret bounded rendering | UT-025 | — | — |
| US-034.EC-2 | Secret-shaped required field guidance | UT-026 | — | — |
| US-035 | One-directional Network bridge | UT-141, UT-150..152 | IT-060 | E2E-027, E2E-028 |
| US-035.EC-1 | Publish without participation | UT-142 | — | E2E-027 |
| US-035.EC-2 | Publish non-terminal rejected | UT-143 | — | E2E-027 |
| US-035.EC-3 | Over-ceiling evidence bounded | UT-144 | IT-060 | — |
| US-035.EC-4 | Publish replay per conversation | UT-163 | — | — |
| US-036 | Call/message lifecycle hooks | UT-145 | IT-056 | — |
| US-036.EC-1 | Hook failure never blocks | UT-145 | — | — |
| US-036.EC-2 | Hook scoping enforced | — | IT-066 | — |
| US-037 | Consented Host API reads | UT-146 | IT-066 | — |
| US-037.EC-1 | Consent denied → zero data | — | IT-066 | — |
| US-037.EC-2 | Reads workspace-scoped | — | IT-066, IT-057 | — |
| US-038 | One operator-caller identity | — | IT-070 | E2E-001 |
| US-038.EC-1 | Concurrent first calls converge | — | IT-070 | — |
| US-038.EC-2 | Workspace deletion cascades | — | IT-058, IT-070 | — |
| `internal/contracts` | registry/validate/normalize/extract/budget/redact | UT-001..028, UT-148, UT-161 | IT-061, IT-072 | — |
| `internal/calls` service | create/settle/await/cancel/mailbox/drain/deadline | UT-029..082, UT-149 | IT-001..044, IT-068..071 | — |
| `[calls]` config | defaults/validate/overlay/tool-surface | UT-083..087 | IT-062 | — |
| CLI verbs | bundles/exit codes/errors | UT-089..095, UT-151, UT-162 | — | E2E-001..011, E2E-029 |
| API handlers | validation/status/projections | UT-096..102, UT-150, UT-154..159 | IT-063, IT-064 | E2E-023..026, E2E-028 |
| Session park/reaper | idle clock/revive/no-reap-with-open-call | UT-103..109 | IT-041, IT-065 | — |
| Hooks + Host API | call family dispatch/consent | UT-145, UT-146 | IT-066 | — |
| Native tools | descriptors/toolsets/digests/availability/spawn deletion | UT-160 | — | — |
| Observability | canonical events per lifecycle path | — | IT-067 | — |
| Web logic | projections/keys/attention/rows | UT-125..136 | — | E2E-015..022 |

## Unit Tests

### internal/contracts (Spec: Implementation Design › Core Interfaces)

- **UT-001** (happy): `Registry.Pin` — object schema in → `Contract{Digest:"sha256:…"}`; identical logical schema with reordered keys pins the same digest (canonicalization).
- **UT-002** (happy): `Registry.Resolve` — pinned digest returns identical canonical bytes.
- **UT-003** (error): `Resolve` unknown digest → typed not-found error.
- **UT-004** (happy): `Validate` — conforming payload → `Verdict{Valid:true}`, zero issues.
- **UT-005** (error): `Pin` with non-object root or invalid schema → typed `call_expect_invalid`-class error naming the schema fault verbatim.
- **UT-006** (error): `Validate` missing required key → `Valid:false`, issue path `$.findings[0].severity`, message verbatim from the engine.
- **UT-007** (boundary): `Validate` payload `{"output": <valid>}` → `Valid:true, Unwrapped:true`; nested double-wrap stays invalid.
- **UT-008** (happy): repair-prompt builder — renders issues verbatim, caps at 10 with `"(+N more)"`, never re-pastes the schema.
- **UT-009** (error): compile failure of a stored schema classifies as contract-fault (not child-fault) and does not consume `repair_attempts`.
- **UT-010** (boundary): 25 issues → deterministic first-10 selection, stable order across runs.
- **UT-011** (happy): budget `store` overflow — 300 KiB payload with 256 KiB budget stores whole, marks overflow, preview bounded.
- **UT-012** (error): budget `reject` overflow → `call_result_over_budget` with declared limit in the message.
- **UT-013** (error): `ResolveBudget` with an override > `max_budget` → rejected at call admission naming the allowed range; the registry row is never touched.
- **UT-014** (boundary): payload exactly at `MaxBytes` accepted; +1 byte triggers overflow strategy.
- **UT-015** (happy): `ExtractCandidate` — fenced block, prose-wrapped, and brace-balanced candidates all found.
- **UT-016** (ordering): two candidates, later invalid-then-earlier-valid vs later-valid — newest valid wins.
- **UT-017** (happy): loop settle adapter — `completedRunAgentOutputFailure` equivalent through contracts produces identical verdict + `invalid_output` mapping as pipeline 1 (parity fixture).
- **UT-018** (error): declared contract on a payload-less node class → lint error from the single resolver.
- **UT-019** (happy): the ONE resolver — same node fixture yields one schema for lint, review, and amendment callers (three-way parity).
- **UT-020** (happy): ask/review `ValidateWaitPayload` adapter produces byte-identical acceptance vs current behavior on the golden fixtures.
- **UT-021** (state): explicit result kinds — sentinel strings (`branch:false`, `error_routed:x`) no longer parse as payloads; readers get typed kind, `outputValue` string-fallback deleted.
- **UT-022** (happy): `RedactPreservingContract` — claim-token value inside a free-text field scrubbed to hash form, contract still validates.
- **UT-023** (error): redaction that would break validity (token in an enum-bound field) → typed rejection telling the child what to remove.
- **UT-024** (happy): key-denylist (`apikey`, `accesstoken`, …) fields scrubbed; `*_hash` forms pass untouched.
- **UT-025** (boundary): secret split across two fields — bounded untrusted rendering caps and labels; scan best-effort documented result.
- **UT-026** (error): contract authoring with a required secret-shaped field pattern → authoring-time guidance error.
- **UT-027** (concurrency): compiled-cache — 100 goroutines validating one digest compile exactly once (counter probe); distinct digests get distinct entries.
- **UT-028** (happy): `x-compozy-kind` walk relocated — agent-name annotation resolves against a fake catalog; unknown entity → typed issue (behavior parity with loop's walk).
- **UT-148** (happy/error): `SanitizeText` — clean text passes unchanged; secret-shaped values are hash-redacted with redaction records; unredactable content (structurally all-secret) returns `reject` with a fixed typed reason; runs before any downstream consumer.

### internal/calls — Service (Spec: System Architecture / Core Interfaces)

- **UT-029** (happy): `Create` with `Target{Agent:"reviewer"}` — pins contract, writes the `calls` row as `queued` in the admission transaction, and the authoritative activation claim transitions it to `running` (narrowed atoms reach `SessionInvoker.SpawnChild`); returns record with `call_id`, `child_session_id`.
- **UT-030** (happy): `Create` without `Expect` → `expect_digest` NULL; record marked uncontracted.
- **UT-031** (happy): `Create` pins prompt to `prompt_ref` blob; long prompt round-trips exactly.
- **UT-032** (happy): `Create` inherits definition runtime; explicit `Runtime` override recorded.
- **UT-033** (happy): `IdleTTL` zero → config default applied at call time.
- **UT-034** (error): unknown agent → `call_agent_unknown` carrying the current roster (names + descriptions).
- **UT-035** (error): invalid `Expect` → `call_expect_invalid` before any spawn (SessionInvoker never called).
- **UT-036** (error): empty/whitespace prompt → typed error, no side effects.
- **UT-037** (error): live-children cap reached → typed cap error naming cap and current count.
- **UT-038** (error): narrowing atom absent from caller's set → `call_widening_rejected` listing exactly the widening atoms.
- **UT-039** (state): two keyless identical `Create`s → two rows, two spawns.
- **UT-040** (happy): `Create` with `Target{SessionID}` — no spawn; delivery to existing child; parked child triggers `Revive`.
- **UT-041** (error): expired target → `call_target_expired` with expiry timestamp; unknown → `call_target_not_found` (distinct codes).
- **UT-042** (error): target outside caller lineage and no grant → typed permission denial.
- **UT-043** (happy): `CreateBatch` 3 items → 3 rows sharing `batch_id`, per-item outcomes ordered.
- **UT-044** (happy): batch per-item `expect` pins distinct digests.
- **UT-045** (error): batch of `max_batch+1` → `call_batch_over_cap`; zero rows written.
- **UT-046** (error): empty batch → typed error.
- **UT-047** (error): item 2 unknown agent → item outcome error; items 1,3 accepted (isolation).
- **UT-048** (error): cap crossed at item N → items ≥N rejected with cap reason; earlier stand.
- **UT-049** (boundary): 800-item batch rejected in <100 ms (validation-only path).
- **UT-050** (idempotency): same key + same payload → original record, `replayed:true`; completed original returns terminal + result ref immediately.
- **UT-051** (error): same key + different payload → `call_idempotency_conflict` naming the original call id.
- **UT-052** (happy): `Cancel` running call → invoker stop, state `canceled`, actor + reason recorded.
- **UT-053** (state): cancel on terminal → no-op response naming current terminal; repeated cancel idempotent.
- **UT-054** (state): cancel queued batch item → `canceled` without any child turn started.
- **UT-055** (happy): `Await` on settled call returns terminal immediately with preview.
- **UT-056** (happy): `Await` multi-id partial — one settles inside window → `settled:[…], pending:[…], outcome:"partial", resume` token.
- **UT-057** (error): await unknown id → typed error immediately.
- **UT-058** (boundary): `timeout_ms` above max clamps to documented bound; response reflects clamp.
- **UT-059** (happy): `Return` valid payload — returns `Settlement{State: completed, Verdict: returned}` with a resolvable result reference and provenance populated (atomicity is owned by IT-021, not by this case).
- **UT-060** (state): second `Return` → `call_already_settled`; stored result unchanged.
- **UT-061** (error): `Return` from session with no active bound call → `call_return_unbound`.
- **UT-062** (error): first invalid `Return` → repair delivery with issues verbatim, `repair_attempts=1`, state stays running.
- **UT-063** (state): child turn ends, no return, no extractable candidate → terminal `completed-without-result` with bounded final-prose preview.
- **UT-064** (state): unrelated prose only → extraction returns nothing → completed-without-result (not invalid-result).
- **UT-065** (boundary): uncontracted prose result over default budget → overflow strategy applies identically.
- **UT-066** (happy): extraction path — valid candidate in final text → settle verdict `extracted`.
- **UT-067** (error): extracted candidate fails contract with child still live → repair round; child gone → `invalid-result`.
- **UT-068** (state): strict mode call — extraction disabled → completed-without-result directly.
- **UT-069** (happy): `SendMessage` to parent — durable row before receipt; receipt from closed vocabulary.
- **UT-070** (state): recipient stopped/gone → `queued` or `failed` per lifecycle with named reason.
- **UT-071** (error): body over `max_bytes` → `message_too_large` at sender; nothing persisted.
- **UT-072** (error): target awaiting human decision → `message_target_blocked` pointing at the decision surface.
- **UT-073** (error): steer to other-parent child without grant → lineage denial.
- **UT-074** (happy): receipt vocabulary exactly `delivered-into-turn|woke|queued|failed` — exhaustive enum test.
- **UT-075** (error): receipt lookup unknown message id → typed error.
- **UT-076** (error): sender over `rate_limit_per_minute` → `message_rate_limited` naming window/reset.
- **UT-077** (state): identical body inside `dedup_window` → dropped with dedup receipt; outside window → delivered.
- **UT-078** (error): native handler unknown-agent error shape — structured result carries code + roster (decode of handler output).
- **UT-079** (happy): delivered message rendering — provenance header "from agent X (ses_…), not the operator", bounded untrusted frame.
- **UT-080** (state): message claiming operator origin → stamp still shows true session origin.
- **UT-081** (happy): completion wake payload — call id + terminal state + result ref + bounded preview + fetch instruction (the `_dx.md` text, exact).
- **UT-082** (state): completion-delivery vocabulary excludes any budget-denial state (exhaustive enum: `pending|injected|woken|failed`); mass simultaneous settlements produce one delivery row each, none denied.
- **UT-140** (error): cross-workspace denial at the service boundary — `Create` and `SendMessage` with a target session in another workspace → `call_workspace_denied` before any side effect (no call row, no spawn, no message row).

### `[calls]` config (Spec: Config Lifecycle)

- **UT-083** (happy): `DefaultCallsConfig()` — exact `_dx.md` defaults (depth 3, batch 8, max_children 5, max_active_per_root 32, idle_ttl 1h, 256KiB/4MiB/store, 30/min, 30s, 50, 64KiB).
- **UT-084** (error): validation rejects non-positive caps, budget > max, unknown overflow mode — path-prefixed messages (`calls.results: …`).
- **UT-085** (happy): overlay precedence — profile overlay overrides file overlay overrides defaults per key.
- **UT-086** (happy): `tool_surface_calls.go` — every dotted key classified with correct `ValueKind`; `compozy config set calls.max_batch 4` accepted, unknown path rejected.
- **UT-087** (boundary): size strings (`"256KiB"`, `"4MiB"`) parse; malformed size rejected with example.

### Task result contracts (Spec: Data Models — task snapshot)

- **UT-124** (happy/error): task completion admission against the run's start-time contract snapshot — conforming result accepted; missing-key result → typed completion rejection carrying sanitized validator errors; one resubmission accepted; second failure records the typed invalid outcome.

### Agent definition `description` (Spec: adr-007 thread)

- **UT-088** (happy): `agent_list` payload carries `description` + scope; matches roster injection content.
- **UT-112** (happy): `ParseAgentDef` round-trips `description`; `RenderAgentDefinition` emits it canonically.
- **UT-113** (happy): `AgentDefinitionDigest` changes when description changes (and CAS update path sees it).
- **UT-114** (happy): definition without description valid; empty string in payloads.
- **UT-115** (error): description over the length bound → validation error naming the bound.
- **UT-116** (happy): description with markdown/injection text renders inert (escaped) in roster string.
- **UT-117** (happy): roster renderer — shadow-aware (workspace wins), format `name — description`, deterministic order.
- **UT-118** (boundary): 50 definitions → capped at 32 entries/120 chars each + overflow line pointing to `agent_list`.
- **UT-119** (state): zero definitions → roster text says so and points to creation.
- **UT-123** (happy): list distinguishes `shadowed` entries from active ones.

### Depth & governance (Spec: adr-008)

- **UT-120** (happy): depth-2 child's toolset still contains `agent_call`; depth-3 child's toolset omits it (strip at wall).
- **UT-121** (happy): child context states literal remaining depth ("You may delegate 1 more level" / "You cannot delegate further").
- **UT-122** (state): `max_depth` raise applies to new spawns; in-flight lineage budgets unchanged.
- **UT-137** (happy): narrowing composes — grandchild atoms validated against child's (already narrowed) set.
- **UT-138** (error): hook mutation that widens → rejected exactly like caller widening (post-hook re-validation).
- **UT-139** (state): caller's effective set changed between calls → second call validates against current set.
- **UT-110** (happy): drain planner — parent terminal enumerates open calls of the governed subtree, closes each with parent-terminal reason.
- **UT-111** (happy): subtree walk covers depth 3 and survives a forged cycle (visited-set guard).

### Session park/reaper (Spec: session integration)

- **UT-104** (state): call settle with empty queue → park transition: runtime stop requested, `parked_at` set, `idle_expires_at = now + idle_ttl`.
- **UT-105** (state): new call/message on parked → `idle_expires_at` cleared (NULL while in flight).
- **UT-106** (state): revive over the root execution budget → revival queues (activation run created, unclaimed); child stays parked until capacity frees; nothing half-starts and nothing rejects.
- **UT-107** (state): revive with changed definition → child keeps spawn-bound definition; drift flag on record.
- **UT-108** (happy): TTL error payload carries expiry timestamp and fresh-call suggestion.
- **UT-109** (state): `idle_ttl` config change → existing parked children keep armed clocks; new parks use new value.
- **UT-103** (state): reaper candidate query excludes sessions with open calls (invariant 11) — fixture with running call never classified.

### CLI (Spec: Agent Manageability Plan; `_dx.md` CLI)

- **UT-089** (happy): `compozy call reviewer "…" --expect @f.json` — human output rows exactly as `_dx.md`; `-o json` carries the full record; exit 0.
- **UT-090** (error): unknown agent → stderr structured error `call_agent_unknown` + roster + `try:` line; exit 2.
- **UT-091** (happy): `call list` four formats via `listBundle` (json array, jsonl rows, human table columns CALL/AGENT/CHILD/STATE/AGE/RESULT, toon).
- **UT-092** (happy): `call await --timeout 300s` timeout → resume token printed; exit 3.
- **UT-093** (happy): `call result` prints whole payload bytes unmodified.
- **UT-094** (happy): `message send` / `message list --session -o jsonl` shapes per `_dx.md`.
- **UT-095** (error): `--expect` malformed JSON file → config-invalid exit code; message names the parse error.

### API handlers (Spec: API Endpoints)

- **UT-096** (happy): `POST /calls` 201 body exactly `{call_id, child_session_id, state, replayed, idle_expires_at:null}`.
- **UT-097** (error): unknown agent 404 `call_agent_unknown` with `available` roster array in body.
- **UT-098** (error): expired target 410; conflict 409; widening/batch-cap/expect-invalid 422; cross-workspace 403 — each with its code in body.
- **UT-099** (happy): batch request → status `200` asserted, body a per-item array with mixed `{call_id}` / `{error}` entries.
- **UT-100** (happy): `GET /calls` cursor pagination — stable ordering, `next_cursor` round-trip, workspace filter enforced in query.
- **UT-101** (happy): `POST /await` 200 both shapes (settled / timeout+resume).
- **UT-102** (happy): `GET /result` returns stored payload verbatim; 409 `call_not_settled` before terminal.
- **UT-154** (error): `POST /messages` 429/413/409 mapped from service errors with codes.

### Web logic (Spec: `_uiux.md` component plan)

- **UT-125** (happy): `agent-comms-tree.ts` projection — calls fixture → tree with depth, states, parked/running/gone partitions; cycle guard on forged parent loop.
- **UT-126** (happy): query-key factory hierarchy (`agentCommsKeys.workspace(id).calls(query)`) normalizes undefined segments.
- **UT-127** (happy): call-detail view-model — verdict `extracted` renders provenance chip; `invalid-result` renders both attempts' issues.
- **UT-128** (happy): empty-state resolution via `resolveDataSurfaceState` (no calls → empty, error → error, loading → loading).
- **UT-129** (boundary): 200-node tree → collapsed rendering buckets; row virtualization threshold respected.
- **UT-130** (happy): inspector Calls tab registration — `InspectorTabId` union + testid map complete (`satisfies` exhaustive).
- **UT-131** (boundary): calls panel pagination — count from summary projection, not page length.
- **UT-132** (state): pruned counterpart → row renders ids, link disabled-absent.
- **UT-133** (happy): `deriveAttentionBadges` with `calls` counts — badge only when descriptor declares it; needs-you causes mapped (invalid-result, completed-without-result, blocked).
- **UT-134** (state): auto-resolved cause → count drops without any dismissal action.
- **UT-135** (happy): coalescing — 5 failures in one tree → one row with count, jump target tree root.
- **UT-136** (happy): cost line uses `describeCost()` output verbatim (`≈`, `Unavailable`, `—` cases).

### Hooks + Host API (Spec: Extensibility Integration Plan)

- **UT-145** (happy): `call` hook family — all seven events registered in catalog, `Validate()` passes, dispatch fires at create/settle/cancel/publish/message-sent/message-delivered/subtree-drained transitions with sanitized typed payloads (fake sink assertion).
- **UT-146** (happy): Host API consent — `calls/list|get|result`, `messages/list` present in generated registry with `calls:read` contracts; permission resolution grants/denies per declared list.

### Network bridge (Spec: adr-005)

- **UT-141** (happy): `Service.Publish` on a completed call → `PublishBridge` receives bounded result evidence + call identity; receipt carries the network message id; attribution = publisher session.
- **UT-142** (error): publish without Live participation → `call_publish_no_participation`; call record untouched.
- **UT-143** (error): publish eligibility is exactly `state == completed` with a valid result reference — table-driven over every other state (queued, running, invalid-result, completed-without-result, failed, canceled, timeout, expired) → `call_publish_not_settled` for each.
- **UT-144** (boundary): result over Network ceiling → evidence = preview + reference, envelope under 1 MiB.

### Native tool family registration (Spec: Impact Audit — native tools)

- **UT-160** (happy/error): the calls tool family — every `compozy__agent_call/call_*/agent_message` descriptor validates (risk classes as audited, `additionalProperties:false`, computed schema digests), belongs to `ToolsetIDCalls`, has a binding + availability closure (bijective boot validation passes), `compozy__session_stop` carries `subtree`, and `compozy__session_spawn` is absent from descriptors, toolsets, and bindings (deletion verified).

### Subtree drain surface (Spec: daemon bridges)

- **UT-147** (happy): session-stop handler with `subtree:true` → invokes `Service.DrainSubtree` (fence-first) and returns the drain report (children stopped, calls closed, results preserved counts); `subtree:false` behaves exactly as today.
- **UT-149** (state): `SweepDeadlines` — due-call selection by indexed `deadline_at`; expired queued call → activation run fenced (canceler fake) then settled `timeout`; expired running call → managed-stop then `timeout`; return-vs-deadline and cancel-vs-deadline resolve to exactly one terminal (unit-level interleavings).

### Publish surface (Spec: API Endpoints / Agent Manageability Plan)

- **UT-150** (happy/error): `POST /calls/{id}/publish` handler — 200 body `{network_message_id, published:true}`; 422 `call_publish_no_participation`; 409 `call_publish_not_settled`; 403 `call_workspace_denied` — status AND body asserted for each.
- **UT-151** (happy/error): `compozy call publish` — all four output formats; non-terminal call exits 2 with `call_publish_not_settled` and the `try:` line.
- **UT-152** (happy): native `compozy__call_publish` handler decodes to the structured result `{network_message_id, published}` with bounded preview.
- **UT-153** (withdrawn — reserved during round-2 drafting, never assigned).
- **UT-155** (happy/error): `deadline_seconds` parity — accepted on HTTP create, batch items, and `compozy__agent_call`; converted to absolute `deadline_at` at admission; zero/negative/non-integer → `call_deadline_invalid` (422 / typed tool error); included in idempotency comparison (same key, different deadline → conflict).
- **UT-156** (happy/error): `GET /calls/{call_id}` — 200 with the full record shape; unknown id → 404 `call_target_not_found`; other-workspace id → 403 — status AND body asserted.
- **UT-157** (happy/idempotency): `POST /calls/{call_id}/cancel` — 200 `{state: canceled}`; repeat → 200 with the same terminal (idempotent, never `call_already_settled`).
- **UT-158** (happy): `POST /messages` — 202 `{message_id, delivery}` exact shape from the `_dx.md` payload.
- **UT-159** (happy): `GET /messages?session=…` — filtered page with direction/provenance/delivery fields; workspace-scoped.
- **UT-161** (happy/error): normalization — the `_dx.md` shorthand example pins the SAME digest as its expanded full schema; a value that is neither form → `call_expect_invalid` with the schema error.
- **UT-162** (happy): CLI parsing/output — `compozy session stop --subtree` renders the drain report in all four formats; `--strict`, `--result-budget`, `--result-overflow` parse and reach the create request; malformed budget size → config-invalid exit.
- **UT-163** (idempotency): publish replay — same call + same conversation → recorded `network_message_id` with `published:false` and no second bridge invocation; different conversation → new publication row + new message.

## Integration Tests

### Store transactions (real SQLite, migration chain)

- **IT-001**: migration `00078` extends the canonical fresh/reopen/ahead/integrity/equivalence suite — fresh apply creates fragment-73 tables; reopen idempotent; ahead-of-binary DB refused without mutation; `atlas.sum` integrity passes; declarative-vs-migrated schema equivalence holds; existing migration bytes and `atlas.sum` history are byte-identical to before the change.
- **IT-002**: **admission transaction** — contract registry row + prompt blob + call row + the `call_activation` task_run (created for every child-starting call) all-or-nothing (injected store failure at each step → zero rows remain, including the run). No subprocess work occurs inside the transaction (invoker fake asserts zero spawn attempts before commit); the fast path claims that exact run immediately after commit.
- **IT-003**: per-parent `max_children` is an admission wall that REJECTS: cap read inside the tx (concurrent creates can't both pass a cap of 1); the losing create gets the typed cap error and spawns nothing — never queues.
- **IT-004**: crash after admission commit, before activation → recovery claims the `call_activation` run and starts the child, or settles the call `failed` with a typed spawn reason; a committed call can never sit unstartable with no owner (no call-table scan involved — recovery flows through the single work queue).
- **IT-005**: 10 concurrent same-key creates → exactly one row; 9 replay responses (DB UNIQUE fence).
- **IT-006**: follow-up create against live child → no new session row; second call row shares `child_session_id`.
- **IT-007**: expired child (reaped, `idle_expires_at` past) → 410-class error; missing id → 404-class (distinct store outcomes).
- **IT-008**: two calls queued to one child → delivered sequentially at boundaries; both settle independently with correct results.
- **IT-009**: create racing the reaper on an expiring child → exactly one of delivered-before-reap / `call_target_expired` (loop 50×, no third outcome).
- **IT-010**: batch of 3 with item-2 invalid → two calls persisted + spawned; item-2 zero side effects.
- **IT-011**: admitted calls beyond `max_active_per_root` (the execution budget) stay `queued` with their `call_activation` runs unclaimed; the budget is evaluated inside the claim transaction and runs are claimed only through `task.Service.ClaimNextRun` as capacity frees; the `calls` tables carry no claim/lease columns and no component starts work from a call-row scan — restart mid-queue recovers through the work queue alone.
- **IT-012**: same-key different-payload → conflict; original row byte-identical after.
- **IT-013**: replay after daemon restart returns original (idempotency survives reopen).
- **IT-014**: retention-pruned original + same key → new call, `replayed:false`.
- **IT-015**: cancel running → invoker stop invoked, terminal `canceled`, repeated cancel returns same terminal.
- **IT-016**: cancel racing `Return` — 50 iterations: exactly one terminal wins; loser lands in `superseded_ref`; no lost update (`-race`).
- **IT-017**: child ignoring stop → managed-stop escalation path stops subprocess; call `canceled` with escalation reason.
- **IT-018**: await registered-before-snapshot — settle fired between snapshot and wait → await resolves (no missed edge).
- **IT-019**: await + daemon restart → resume converges on durable terminal; no lost completion.
- **IT-020**: 32 concurrent awaits on one call all resolve; 33rd rejected per wait-cap (extends the session-wait canonical suite).
- **IT-021**: `Return` settlement transaction — validate + blob write + terminal + delivery row atomic; injected blob-write failure rolls back all.
- **IT-022**: double return — second rejected inside tx; delivery row not duplicated.
- **IT-023**: return racing child crash → exactly one durable outcome (completed with valid ref XOR failed with crash reason).
- **IT-024**: invalid → repair delivered → invalid again → terminal `invalid-result` with both issue sets persisted.
- **IT-025**: contracted child ends turn silent → extraction miss → `completed-without-result` with prose preview persisted.
- **IT-026**: TTL-stop mid-work (forced) → `failed` with TTL reason (never completed-without-result).
- **IT-027**: 300 KiB result, `store` overflow → whole blob stored, digest re-verified on read, preview bounded in projections.
- **IT-028**: prose-only valid candidate → settle verdict `extracted`; digest re-verification on read passes.
- **IT-029**: child→parent message — row committed before any notify (crash-after-commit test recovers and delivers).
- **IT-030**: receipt transitions queued→delivered-into-turn recorded with timestamps; sender observes both states.
- **IT-031**: restart with pending deliveries → boot drain delivers exactly once (durable `wake_event_id` dedupe).
- **IT-032**: message sent during child terminal write → deterministic ordering; parent reads message attributed to pre-terminal context.
- **IT-033**: steer to busy child (acpmock long tool) → injected only at boundary; multiple steers preserve order; running tool never interrupted (event-stream assertion).
- **IT-034**: repeated delivery failures → policy terminal `failed` with reason; no infinite retry loop.
- **IT-035**: two sessions in a reply loop → rate limit + dedup + pending cap terminate the loop; every brake engagement observable in receipts/events.
- **IT-036**: 8-item batch completing simultaneously → all completion deliveries land; none dropped as duplicates or rate-limited.
- **IT-037**: delivered peer message cannot resolve a pending permission prompt (prompt still pending after delivery); embedded `/compact` arrives as text (transcript assertion); relayed denied-action request does not alter recipient's permission evaluation.
- **IT-038**: completion wake carries result ref — parent acpmock turn receives wake meta with call id + preview; fetch by id returns full payload.
- **IT-039**: 5 children complete while parent busy → deliveries injected in stable order at next boundaries.
- **IT-040**: delivery rows carry `owner_key`; usage projection counts call activations once (no double-booking against Network rows).
- **IT-041**: park→revive cycle — settle parks (runtime gone, row live), message revives, context preserved (transcript continuity), caps accounted at revive.
- **IT-042**: two concurrent revival triggers → one revival (one activation run); both deliveries land in order; revival over the root budget stays queued and activates exactly once when capacity frees.
- **IT-043**: parent stop with running + parked children → drain: children stopped, open calls `failed(parent-terminal)`, completed results still fetchable.
- **IT-044**: parent crash → recovery drain equals clean-stop drain (same terminal set) across depth-3 subtree.
- **IT-045**: agent create → list round-trip with description over HTTP + tool; shadow scenario (workspace + global same name) lists winner + shadowed.
- **IT-046**: roster in tool description updates after definition add/remove without daemon restart (projection re-render).
- **IT-047**: depth chain root→d1→d2→d3 spawns succeed; d3→d4 rejected `call_depth_exceeded`; d3 toolset lacks the call tool.
- **IT-048**: depth-3 × batch fan-out against governed-root cap → over-cap items queue/reject visibly (named reasons), none silent.
- **IT-049**: concurrent definition update + list → consistent snapshot (no torn read).
- **IT-050**: loop run-agent node with `output_schema` → capture + settle validation both flow through `internal/contracts` (instrumented single-resolver assertion).
- **IT-051**: forced stored-payload corruption between capture and settle → settle demotes to `invalid_output` (the #438 regression, now via contracts).
- **IT-052**: task with `expect` declared via `compozy task create --expect` → conforming completion accepted; missing-key completion rejected typed with one resubmission round; `task update --expect` mid-run → the in-flight run settles under its `task_runs` snapshot (start-time rules), the next run under the new digest.
- **IT-053**: 70 KiB valid task result with 256 KiB contract budget → accepted (the old 64 KiB blanket gone); over-contract-budget behaves per strategy.
- **IT-054**: same contract + payload validated via call path, loop path, task path, ask path → identical verdicts and issue rendering (four-way parity).
- **IT-055**: loop refactor — branch/route/skip control outcomes flow through typed result kinds end to end in the owning loop behavior suite (branch-false, route-not-taken, absorbed-failure fixtures produce the typed kinds and correct template-namespace values); stale readers are caught by compilation and boundary gates, not text assertions.
- **IT-056**: spawn hooks on the call path — `spawn.pre_create` narrowing honored; widening hook mutation rejected post-hook; `call.created` fires after commit.
- **IT-057**: cross-workspace call/message → denied pre-side-effect; list/read/SSE for workspace A never return workspace B rows (seeded both).
- **IT-058**: workspace deletion with parked children + open calls → all records terminal with workspace-removal reason; no dangling addressable rows.
- **IT-059**: sanitize-before-everything — (a) a raw claim token inside a **valid** return payload is hash-redacted at storage; (b) inside an **invalid** payload the validator error, repair prompt, hook payloads, logs, SSE events, and stored rows are swept and show only hash forms (verbatim-from-sanitized); (c) a raw token inside a plain-text **message** (agent→parent and operator→child) is sanitized by `SanitizeText` before dedup hashing, hooks, receipts, events, and persistence — stored body, receipt, hook, log, SSE, and the delivered frame are swept; (d) a redaction-validity conflict returns the fixed typed error with paths, never values.
- **IT-060**: publish completed call to a Live channel → timeline row with evidence + call id; reverse direction impossible (no API accepts channel→call).
- **IT-061**: contracts registry against real store — pin/resolve/dedup (two pins, one row), cache survives across service instances.
- **IT-072**: one schema, two budgets — two calls pin the same schema concurrently with different `result_budget` overrides → one `contract_schemas` row, two independent immutable snapshots; each call's overflow behavior follows its own snapshot; replaying either idempotency key with a different budget → `call_idempotency_conflict`.
- **IT-062**: config file + profile overlay + `compozy config set calls.messages.pending_cap 10` → effective value visible via `config get` and enforced by the service.
- **IT-063**: UDS route parity — every `/calls`/`/messages` route registered on both transports (extends `handlers_test.go` list).
- **IT-064**: OpenAPI document contains the call/message operations with tags; generated TS types compile (codegen-check).
- **IT-065**: reaper sweep with mixed fixture (open-call child, parked-in-window, parked-expired) → only parked-expired reaped.
- **IT-066**: extension host API `calls/list` through a test extension → consent gate enforced (granted vs denied fixture).
- **IT-067**: observability coverage matrix — every call/message lifecycle path emits its canonical event with correlation keys; missing-event fixture fails.
- **IT-068**: `call_activation` queue mechanics against real SQLite — migration 00078 rebuilds the `task_runs` CHECK (taskless `call_activation` inserts succeed; unknown kinds still rejected); exact-kind claim selection returns only call activations to the dispatcher; the governed-root execution budget is evaluated inside the claim transaction (two claims against budget 1 → one wins); `call_activation_runs` side table binds run↔call.
- **IT-069**: deadline authority races — expired queued call: sweep-vs-claim race resolves to exactly one outcome (fenced-then-timeout XOR claimed-then-managed-stop); expired running call settles `timeout` with the result-carrying terminal wake; return arriving during sweep → one terminal, loser recorded superseded; boot after crash mid-sweep reconciles.
- **IT-070**: operator-caller fence — two concurrent first operator calls (CLI + HTTP) in one workspace → exactly one `operator_caller_sessions` row and one owner session, two distinct `actor_*` records; reopen preserves the binding; the bound session is excluded from targeting, liveness caps, and the reaper; workspace deletion cascades it.
- **IT-071**: claim-vs-terminalization race on activation runs — 50 iterations of cancel/drain racing `ClaimNextRun` on the same `call_activation` run → exactly one winner every time (canceled-never-claimed XOR claimed-then-managed-stopped); a terminal call with a claimable run never exists (post-race sweep assertion); boot reconciliation repairs an injected crash window between the two records.

## End-to-End Tests

### Runtime + CLI journeys (acpmock; `_dx.md` transcripts verbatim)

- **E2E-001**: golden path — author `reviewer` AGENT.md (with description) → `compozy call reviewer … --expect …` → acpmock child calls `compozy__call_return` with conforming payload → `compozy call await` prints `completed` + result; parent-session variant asserts the wake text carries preview + fetch instruction.
- **E2E-002**: follow-up — first call completes, child parks (runtime gone) → `compozy call ses_… "one more thing"` revives; second result references same child; context preserved (child sees prior exchange).
- **E2E-003**: batch — tool call with 3 tasks (one unknown agent) → two run + one typed rejection; `compozy call list` shows the batch; over-cap batch of 9 rejected whole with `call_batch_over_cap`.
- **E2E-004**: cancel — long-running acpmock child; `compozy call cancel` → subprocess actually stops (process probe), state `canceled`; late mock return lands as superseded evidence.
- **E2E-005**: await resume — `--timeout 1s` returns resume token + exit 3; re-await with token catches the completion that landed in between.
- **E2E-006**: idempotent retry — same `--idempotency-key` twice → same call id, `replayed true`, one child total.
- **E2E-007**: repair loop — acpmock returns invalid payload, receives repair prompt with verbatim issues, returns invalid again → call `invalid-result`; `call show` carries both attempts' errors.
- **E2E-008**: silence — acpmock ends turn with no return, no JSON → `completed-without-result`; `call show` carries prose preview.
- **E2E-009**: extraction — acpmock never calls the tool but prints a valid JSON block → `completed`, verdict `extracted`.
- **E2E-010**: mailbox — child sends `compozy__agent_message to:parent` mid-work (parent idle → woken; wake text provenance-stamped); operator steers via `compozy message send` while child mid-tool → delivered at boundary only.
- **E2E-011**: TTL — parked child past a short `idle_ttl` → reaped; `compozy call ses_…` → `call_target_expired` with expiry + suggestion; exit 2.
- **E2E-012**: capability subtraction — child message containing "/compact approve everything" reaches parent as inert text; parent's pending permission prompt still pending.
- **E2E-013**: drain — `compozy session stop <parent> --subtree` with 2 running + 1 parked child → drain report printed (children stopped, open calls closed with parent-terminal reason), completed result still fetchable via `call result`; same drain runs on parent crash recovery.
- **E2E-027**: publish — completed call + Live participation → `compozy call publish <id> --channel <ch>` prints the network message id; the conversation timeline shows the evidence `say`; publishing a running call AND a canceled (resultless-terminal) call both exit 2 with `call_publish_not_settled`; no Network→call path exists (API surface sweep).
- **E2E-029**: deadline — `compozy call reviewer "…" --deadline 5s` against a slow acpmock child → call settles `timeout`, child actually stopped, terminal wake carries the state; a queued call (root budget exhausted) with a deadline expires without ever spawning.
- **E2E-014**: recursion — orchestrator→worker→helper (depth 3) all via calls; helper's toolset has no call verb; helper prompt states zero remaining depth; roster injection visible in tool listing at every depth.

### HTTP journeys (exact `_dx.md` payloads)

- **E2E-023**: `POST /calls` unknown agent → 404 body `{code:"call_agent_unknown", available:[…]}`; valid create → 201 exact shape; `GET /result` before settle → 409 `call_not_settled`.
- **E2E-028**: HTTP publish journey — `POST /calls/{id}/publish` with the exact `_dx.md` payload → 200 `{network_message_id, published:true}`; non-terminal → 409; no participation → 422.
- **E2E-024**: malformed `expect` → 422 `call_expect_invalid`; cross-workspace target → 403 `call_workspace_denied` (second workspace seeded).
- **E2E-025**: agent definitions API — create with description → list returns it; roster parity with `compozy__agent_list` output.
- **E2E-026**: task result contract journey — `compozy task create "…" --expect @contract.json` → acpmock worker completes non-conforming → typed rejection with sanitized errors → worker resubmits conforming → accepted; `GET /api/workspaces/{ws}/tasks/{id}` shows `expect_digest` + budget, and the run read shows provenance/preview per the unified regime.

### Browser journeys (Playwright; `_uiux.md` surfaces)

- **E2E-015**: Activity tree — seeded tree (running/completed/invalid-result/parked at depths 1-3) renders live; collapse escalates urgency tone; selecting a row opens call detail.
- **E2E-016**: call detail — completed call shows contract digest, timeline, schema-aware result with preview + full-payload open, cost line (`describeCost` wording); cancel button absent on terminal (affordance absent, not disabled).
- **E2E-017**: inbox — seeded messages show provenance + delivery outcomes; compose to a live child delivers (receipt transitions visible); compose to blocked child shows the typed refusal.
- **E2E-018**: liveness — SSE drop + reconnect resyncs to daemon truth; acting on a just-settled call shows stale-action feedback and refreshes.
- **E2E-019**: session Calls panel — inspector tab lists both directions; timeline shows the completion wake row with call id + preview ("why did this wake").
- **E2E-020**: attention — invalid-result call lights the Agents dock badge + bell needs-you row; opening the tree from the row clears on resolution (no manual dismiss exists).
- **E2E-021**: empty states — fresh workspace: Activity/inbox/roster empty states render teaching copy; zero-definition roster points to creation.
- **E2E-022**: scale — 150-call tree stays navigable (collapse buckets, keyboard traversal ↑↓←→ Enter), counts from summary projection match seeded totals.
- **E2E-030**: roster journey — catalog rows show description, scope badge (workspace/global/shadowed), and live instance counts; the Call action opens the prefilled compose, an invalid contract shows `call_expect_invalid` inline, and submitting creates a call that appears in the Activity tree.
