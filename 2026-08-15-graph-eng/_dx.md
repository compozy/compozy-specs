# Developer Experience: Loop Graph Completion (graph-eng)

Public-surface contract for human interaction, routing, fan-out strategies, and operator time travel on Loops. Companion to `_spec.md` (Part II serves this surface) and `_tests.md` (E2E journeys use these exact invocations).

## Golden Path

Author a loop that asks a person mid-run, gates a risky action behind review, shards work with a quorum, and routes by risk — then answer it, watch it finish, and fork it to try different inputs.

```bash
# 1. Author the loop (file below), publish it
compozy loop create --file rollout.yaml
# → Loop rollout published (version 1)

# 2. Start a run
compozy loop run --name rollout --input service=billing -o json
# → {"run":{"id":"lr_01j9...","status":"running",...},"web_url":"/loop-runs/lr_01j9..."}

# 3. The run parks at the ask node — list what needs you
compozy loop requests
# REQUEST                        RUN         NODE            KIND    AGE   EXPIRES
# ask · Which environments?      lr_01j9…    pick_envs       ask     2m    in 71h

# 4. Answer it
compozy loop respond --run-id lr_01j9 --node pick_envs \
  --payload '{"environments":["dev","staging"]}'
# → answered · run resumed

# 5. The publish step parks for review — approve it with an edit
compozy loop respond --run-id lr_01j9 --node publish \
  --decision edit --payload '{"tag":"v2.4.1","notes":"Hotfix rollup"}' \
  --note "fixed the tag prefix"
# → answered (edit) · publish executing with edited arguments

# 6. Run completes partial (quorum met, 1 shard failed) — partiality is run-level truth
compozy loop status --run-id lr_01j9 -o json | jq '{status: .run.status, completion: .run.completion_state}'
# → {"status":"done","completion":"partial"}

# 7. Re-execute the flaky shard — a new generation opens — then compare the two
compozy loop rerun --run-id lr_01j9 --from-node shard_verify --reason "flaky infra"
# → rerun · lr_01j9 · generation 2 opened (origin: operator_rerun, parent: 1)
compozy loop diff --run-id lr_01j9 --generation 1 --against-generation 2

# 8. Fork from generation 2 with different inputs; the original is untouched
#    (the fork seeds its previous/best baseline from gen 2 and runs the full body)
compozy loop fork --run-id lr_01j9 --generation 2 --input service=payments -o json
# → {"run":{"id":"lr_01ja...","forked_from":{"run_id":"lr_01j9...","generation":2}},...}
```

## YAML

The complete definition the golden path publishes. Lives wherever loop definitions live today (workspace `.compozy/loops/<name>/loop.yaml` or an extension kit); loaded by `compozy loop create --file` / the visual editor.

```yaml
apiVersion: compozy.loop/v1
kind: Loop
meta:
  name: rollout
  description: Roll a service out with a human picking environments and reviewing the publish.

concurrency: forbid

inputs:
  service: { type: string, required: true }

contract:
  goal: "Roll out {{ .inputs.service }} to the environments a person picks."
  definition_of_done: "Publish approved and every selected environment verified."
  iteration_cap: 3
  no_progress: { window: 2 } # hash_fields is gone — an unknown-parameter lint error if present
  budget: { tokens: 0, wall_clock_sec: 0, on_exceeded: halt }
  terminal_states: [done, no-op, blocked, failed, exhausted, stalled, canceled]

graph:
  nodes:
    - id: service_input
      class: source
      kind: input
      input_ref: service

    # ── ask: structured input from a person ─────────────────────────────
    - id: pick_envs
      class: control
      kind: ask
      params:
        prompt: "Which environments should the {{ .inputs.service }} rollout include?"
        context:
          service: "{{ .inputs.service }}"
        expect: # required — what a valid answer looks like
          type: object
          required: [environments]
          properties:
            environments:
              type: array
              minItems: 1
              items: { enum: [dev, staging, prod] }
        responders:
          agents: deny # default; `allow` opts agents in for this node
        expires:
          after: 72h
          escalate:
            - emit: { kind: rollout.request_overdue }
          route: halt_rollout

    - id: classify
      class: action
      kind: run-agent
      params:
        agent: risk_analyst
        prompt: "Classify rollout risk for {{ .inputs.service }} across {{ .nodes.pick_envs.output.environments | toJson }}."
        output_schema:
          type: object
          required: [risk]
          properties:
            risk: { enum: [low, medium, high] }

    # ── route: exclusive multi-way routing with a mandatory default ─────
    - id: triage
      class: control
      kind: route
      routes:
        - when: 'nodes.classify.output.risk == "high"'
          to: deep_check
        - when: 'nodes.classify.output.risk == "medium"'
          to: standard_check
      default: quick_check # required — a router without default does not compile

    - id: deep_check
      class: action
      kind: run-agent
      params: { agent: sre, prompt: "Deep-verify the rollout plan." }
    - id: standard_check
      class: action
      kind: run-agent
      params: { agent: sre, prompt: "Verify the rollout plan." }
    - id: quick_check
      class: action
      kind: transform
      params:
        map:
          verified: { value: true }

    # ── fan-out with a quorum strategy and named iteration variables ────
    - id: shard_verify
      class: control
      kind: fan-out
      collection: "{{ .nodes.pick_envs.output.environments }}"
      bind_as: env # `.env` instead of `.item` inside the body
      index_as: env_index
      batch_size: 1
      max_parallel: 2
      max_fan_out: 200 # any bound; the old 64 ceiling is gone
      strategy:
        kind: best_effort
        threshold: 66% # or `count: 2`
        missing: acceptable # required declaration: missing branches are not mandatory coverage
      body:
        nodes:
          - id: verify_env
            class: action
            kind: run-agent
            params:
              agent: sre
              prompt: "Verify {{ .inputs.service }} in {{ .env }}."
        edges: []

    - id: collect_verify
      class: control
      kind: collect

    # ── review: a person approves/edits the action before it executes ───
    - id: publish
      class: action
      kind: compozy__release_publish
      params:
        tag: "{{ .nodes.classify.output.risk }}-{{ .inputs.service }}"
        notes: "Rollout for {{ .inputs.service }}"
      review:
        decisions: [approve, edit, reject, respond] # default when omitted: [approve, reject]
        when: 'nodes.classify.output.risk != "low"' # optional — review only when it matches
        prompt: "Publishing {{ .inputs.service }} — check the tag."
        responders: { agents: deny }
        on_reject: { route: halt_rollout } # omitted → the node fails as a quality rejection
        expires: { after: 24h, route: halt_rollout }

    - id: halt_rollout
      class: action
      kind: transform
      params:
        map:
          halted: { value: true }

  edges:
    - { from: service_input, to: pick_envs }
    - { from: pick_envs, to: classify }
    - { from: classify, to: triage }
    - { from: triage, to: deep_check }
    - { from: triage, to: standard_check }
    - { from: triage, to: quick_check }
    - { from: deep_check, to: shard_verify }
    - { from: standard_check, to: shard_verify }
    - { from: quick_check, to: shard_verify }
    - { from: shard_verify, to: collect_verify }
    - { from: collect_verify, to: publish }
    - { from: pick_envs, to: halt_rollout }

start:
  - { kind: manual }
  - { kind: cli }
  - { kind: http }
  - { kind: native_tool }
```

What the author can now reference:

- `nodes.pick_envs.output.environments` — the validated human answer (compile-checked against `expect`).
- `nodes.shard_verify.progress.success_rate` — live fan-out progress, readable in any condition; inside the body the bare alias `progress.*` works.
- `nodes.collect_verify.status` — `succeeded`, `partial`, or `failed`; `nodes.collect_verify.output` carries `{total, succeeded, failed, canceled, coverage_rate, partial}`.
- Route decisions and their causes appear in run history; skipped routes read "not taken", never "failed".
- Gate verdicts can route too: `on_result: { fail: { route: remediate } }` on any gate selects a pre-declared forward route. String actions (`continue`, `revise`, `halt`, `escalate`, `done`, `next_generation`) still work; the old `branch` action string is **removed** — using it is a lint error pointing at the `{ route: ... }` form.
- Conditions that break at evaluation: `branch`/`route`/`filter` conditions fail the node as a routable authoring failure; a broken `contract.stop_when` exits the loop with a diagnostic instead of iterating forever. Override per node with `on_eval_error: fail | exit`; for the contract's stop condition use the object form — `stop_when: { expr: "…", on_eval_error: fail }` (the plain string keeps the fail-open default).

## CLI

### `compozy loop requests` — list pending human requests

```bash
compozy loop requests --workspace .
```

```
REQUEST                          RUN        NODE       KIND    STATE     AGE   EXPIRES
Which environments should the…   lr_01j9…   pick_envs  ask     pending   2m    in 71h
Publishing billing — check the…  lr_01j9…   publish    review  pending   1m    in 23h
```

```bash
compozy loop requests -o json
```

```json
{
  "items": [
    {
      "loop_run_id": "lr_01j9...",
      "loop_name": "rollout",
      "node_id": "pick_envs",
      "item_index": 0,
      "kind": "ask",
      "state": "pending",
      "prompt": "Which environments should the billing rollout include?",
      "context": { "service": "billing" },
      "expect": { "type": "object", "required": ["environments"] },
      "decisions": ["respond"],
      "opened_at": "2026-08-16T14:02:11Z",
      "expires_at": "2026-08-19T14:02:11Z"
    },
    {
      "loop_run_id": "lr_01j9...",
      "node_id": "publish",
      "kind": "review",
      "state": "pending",
      "proposed_preview": { "tag": "high-billing", "notes": "Rollout for billing" },
      "decisions": ["approve", "edit", "reject", "respond"],
      "expires_at": "2026-08-17T14:09:40Z"
    }
  ],
  "aggregates": { "pending": 2 },
  "next_cursor": ""
}
```

`aggregates.pending` is the daemon-computed count the OS bell composes with the session attention summary (it never counts rows client-side). Inline fields are redacted, bounded previews (`context`, `proposed_preview`); the full redacted context is fetchable per request (its content-addressed ref rides the item). `state=resolved` is the closed union `answered | expired | canceled`. Ordering is stable: pending by soonest expiry (then opened time), resolved by most recent resolution; cursors stay stable across pages.

Flags: `--run-id` (one run only), `--state pending|resolved` (default `pending`), `--cursor`, `--limit` (1..200), `--workspace`. Exit 0.

### `compozy loop request` — one request in full

```bash
compozy loop request --run-id lr_01j9 --node pick_envs -o json
```

```json
{
  "loop_run_id": "lr_01j9...",
  "node_id": "pick_envs",
  "item_index": 0,
  "kind": "ask",
  "state": "pending",
  "prompt": "Which environments should the billing rollout include?",
  "context": { "service": "billing", "plan": "…the full redacted context, not the preview…" },
  "expect": { "type": "object", "required": ["environments"] },
  "decisions": ["respond"],
  "opened_at": "2026-08-16T14:02:11Z",
  "expires_at": "2026-08-19T14:02:11Z"
}
```

The detail read returns the **full redacted context** (what the list truncates to a preview), the per-decision schemas, and resolution provenance once resolved — this is how a responder (human or agent) always reaches everything US-007 promises. Unknown request → deterministic not-found. `--item` addresses one lane.

### `compozy loop respond` — answer an ask or decide a review

```bash
# Answer an ask node (the only decision for ask is the answer itself)
compozy loop respond --run-id lr_01j9 --node pick_envs \
  --payload '{"environments":["dev","staging"]}'
```

```
answered · pick_envs · run lr_01j9 resumed
```

```bash
# Review decisions
compozy loop respond --run-id lr_01j9 --node publish --decision approve
compozy loop respond --run-id lr_01j9 --node publish --decision edit \
  --payload '{"tag":"v2.4.1","notes":"Hotfix rollup"}' --note "fixed tag"
compozy loop respond --run-id lr_01j9 --node publish --decision reject --note "wrong week"
compozy loop respond --run-id lr_01j9 --node publish --decision respond \
  --payload '{"release_url":"https://github.com/acme/billing/releases/v2.4.0"}'
```

```bash
compozy loop respond --run-id lr_01j9 --node publish --decision edit \
  --payload '{"tag":"v2.4.1"}' -o json
```

```json
{
  "ok": true,
  "run_id": "lr_01j9...",
  "node_id": "publish",
  "decision": "edit",
  "state": "answered",
  "provenance": { "actor_kind": "operator", "actor_id": "pedro", "answered_at": "2026-08-16T14:11:03Z" }
}
```

Failure the responder will actually hit — answer does not match `expect`:

```bash
compozy loop respond --run-id lr_01j9 --node pick_envs --payload '{"environments":[]}'
```

```
Error: request_validation_failed: answer does not match the node's expected shape
  environments: minItems: array must have at least 1 items
```

Exit 1. Already answered → `request_already_answered` (exit 1, idempotent — the recorded answer is returned). Expired → `request_expired`. Canceled with its node/run → `request_canceled`. Agent answering without node opt-in → `respond_not_permitted`. The starting agent (or any agent in its spawn chain) answering its own run → `respond_self_denied`. Flags: `--item` for a fan-out lane; `--decision` (`approve|edit|reject|respond`; ask nodes take no `--decision`); `--payload` must be valid JSON.

### `compozy loop node amend` — correct a parked node's output

```bash
compozy loop node amend --run-id lr_01j9 --node classify \
  --payload '{"risk":"medium"}' --reason "analyst over-rated it"
```

```
amended · classify · original preserved in history
```

Amend targets a cell that already **has** a settled output and is paused/parked (pause first for a succeeded node); it never re-runs consumers — pair it with `loop rerun --from-node` when downstream already consumed the bad value. Amendments are an **append-only overlay**: the recorded output is never rewritten; the corrected value becomes the cell's *effective* output (what resume and downstream read), and history/diff show both, with the amender's identity, reason, and timestamp.

Rules surfaced as errors: node not parked/paused → `amend_not_parked`; cell has no output yet → `amend_no_output`; node has no declared output shape → `amend_schema_missing`; payload fails the shape → `request_validation_failed`. Exit 1 on each. `--item` addresses one fan-out lane.

`--item` also lands on `compozy loop node cancel|kill|pause|resume` (and their routes/tools via `item_index`), addressing a single fan-out lane — the per-lane cancellation the completion strategies use, exposed as an operator verb.

### `compozy loop diff` — compare generations or runs

```bash
compozy loop diff --run-id lr_01j9 --generation 1 --against-generation 2
```

```
rollout · lr_01j9 · generation 1 → 2 (origin: gate_revise)

CHANGED  classify        output.risk: "high" → "medium"
CHANGED  triage          route: deep_check → standard_check (matched: risk == "medium")
RERUN    standard_check  succeeded (was: not taken)
SKIPPED  deep_check      not taken (was: succeeded)
CARRIED  pick_envs       unchanged (carried forward)
VERDICT  quality         approved (was: rejected · 2 blocking issues resolved)
```

```bash
compozy loop diff --run-id lr_01j9 --against-run lr_01ja -o json
```

```json
{
  "kind": "run",
  "base": { "run_id": "lr_01j9...", "generation": 2 },
  "against": { "run_id": "lr_01ja...", "generation": 1 },
  "inputs": [{ "key": "service", "base": "billing", "against": "payments" }],
  "nodes": [
    { "node_id": "classify", "change": "changed", "field": "output.risk", "base": "medium", "against": "low" },
    { "node_id": "deep_check", "change": "skipped", "cause": "route_not_taken" }
  ],
  "terminal": { "base": "done", "against": "running" }
}
```

Cross-loop diff → `diff_cross_loop` (exit 1). Large outputs summarize with sizes and content hashes. Diff is read-only; no capability required.

### `compozy loop rerun` — re-execute from a healthy node

```bash
compozy loop rerun --run-id lr_01j9 --from-node shard_verify --reason "flaky infra"
```

```
rerun · lr_01j9 · generation 3 opened (origin: operator_rerun, parent: 2)
rerun set: shard_verify, collect_verify, publish (+2 dependents) · carried: 5 nodes
```

Guards: node still parked/pending → `rerun_node_unsettled`; run mid-generation → `rerun_busy`. Idempotency identifies intent: pass `--request-id` and a retry of the same request returns the already-created generation; reusing a `--request-id` with different arguments → `timetravel_key_reuse`. Without a key, every acknowledged call is a **fresh operation** (two intentional identical reruns are two operations); rapid duplicates are absorbed by `rerun_busy` while the previous generation is in flight. `--item` reruns one lane.

### `compozy loop fork` — new linked run from a historical generation

```bash
compozy loop fork --run-id lr_01j9 --generation 2 --input service=payments -o json
```

```json
{
  "run": {
    "id": "lr_01ja...",
    "loop_name": "rollout",
    "status": "queued",
    "forked_from": { "run_id": "lr_01j9...", "generation": 2 }
  },
  "web_url": "/loop-runs/lr_01ja..."
}
```

`--input` overrides declared inputs (repeatable, JSON values supported) under normal input validation. The fork seeds its `previous.*`/`best.*` baseline from the chosen generation's outputs and executes the **full body** as its first generation — outputs are context, never pre-completed work, so a changed input can't coexist with stale results; no override = a fresh attempt from the same baseline. Missing generation (or a source outside your workspace) → `fork_generation_unknown`. `--request-id` works exactly as on rerun. Fork never touches the source run; both runs list each other (`forked_from` / `forks[]` in `loop status`). Rerun and fork require the `loops.timetravel` capability for agents (diff is an ungated workspace read); an agent cannot rerun/fork its own still-executing run → `timetravel_self_denied`.

## HTTP / UDS API

All routes workspace-scoped under `/api/workspaces/:workspace_id`, matching existing loop routes. Every lifecycle verb returns the shared mutation response shape with provenance.

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/loop-requests?state=pending&run_id=&cursor=&limit=` | Pending/resolved human requests inventory (previews) |
| GET | `/loop-runs/:run_id/nodes/:node_id/request?item_index=` | One request in full (full redacted context + schemas + provenance) |
| POST | `/loop-runs/:run_id/nodes/:node_id/respond` | Answer an ask / decide a review |
| POST | `/loop-runs/:run_id/nodes/:node_id/amend` | Correct a parked node's output |
| GET | `/loop-runs/:run_id/diff?generation=&against_generation=` | Generation diff |
| GET | `/loop-runs/:run_id/diff?against_run=` | Run diff (same loop) |
| POST | `/loop-runs/:run_id/rerun` | Open an operator rerun generation |
| POST | `/loop-runs/:run_id/fork` | Start a new linked run from a generation |

### `POST /loop-runs/:run_id/nodes/:node_id/respond`

```json
{ "decision": "edit", "payload": { "tag": "v2.4.1", "notes": "Hotfix rollup" }, "note": "fixed tag", "item_index": 0 }
```

→ `200`

```json
{
  "ok": true,
  "run_id": "lr_01j9...",
  "node_id": "publish",
  "decision": "edit",
  "state": "answered",
  "provenance": { "actor_kind": "operator", "actor_id": "pedro", "answered_at": "2026-08-16T14:11:03Z" }
}
```

→ `422` when the payload fails the node's expected shape:

```json
{ "error": "request_validation_failed", "details": { "environments": "minItems: array must have at least 1 items" } }
```

→ `409` `request_already_answered` (body carries the recorded decision) · `410` `request_expired` · `403` `respond_not_permitted` / `respond_self_denied`.

### `POST /loop-runs/:run_id/rerun`

```json
{ "from_node": "shard_verify", "item_index": null, "reason": "flaky infra", "request_id": "rerun-shard-1" }
```

→ `200`

```json
{
  "ok": true,
  "run_id": "lr_01j9...",
  "generation": 3,
  "parent_generation": 2,
  "origin": "operator_rerun",
  "rerun_nodes": ["shard_verify", "collect_verify", "publish"]
}
```

→ `409` `rerun_busy` · `422` `rerun_node_unsettled`.

### `POST /loop-runs/:run_id/fork`

```json
{ "generation": 2, "inputs": { "service": "payments" }, "reason": "what-if", "request_id": "fork-payments-1" }
```

→ `200` — the same response shape as starting a run, plus lineage:

```json
{ "run": { "id": "lr_01ja...", "status": "queued", "forked_from": { "run_id": "lr_01j9...", "generation": 2 } }, "web_url": "/loop-runs/lr_01ja..." }
```

→ `404` `fork_generation_unknown` · `422` input validation errors (same shape as run start).

### Run detail additions (`GET /loop-runs/:run_id`)

- `run.completion_state` — `complete | partial`, the run-boundary partiality truth (a partial quorum result never reads as fully complete anywhere).
- `run.forked_from { run_id, generation }` and `run.forks [{ run_id, generation, created_at }]`.
- `requests[]` — this run's human requests with state, previews, and provenance (`answered_at` on resolutions).
- `amendments[]` — every output amendment (node, lane, sequence, redacted original/amended, actor, reason, timestamp); the diff view marks amended cells from the same projection.
- Generation payloads carry route decisions (`route_causes`) and strategy settlements; collect outputs expose `{total, succeeded, failed, canceled, coverage_rate, partial}`.
- New event kinds on the SSE stream (`/loop-runs/:run_id/events`): `request_opened`, `request_answered`, `request_expired`, `request_canceled`, `node_amended`, `route_taken`, `branch_pruned`, `run_forked`. Existing replay semantics unchanged.

## config.toml

```toml
[loops.defaults.delivery.requests]
expire_after = "" # default expiry for ask/review nodes without an explicit `expires` ("" = park indefinitely, matching approvals)

[loops.defaults.watch.requests]
expire_after = ""
```

- `loops.defaults.<kind>.requests.expire_after` — seeds `expires.after` for human requests that declare none; per-node `expires` always wins.
- `loops.defaults.<kind>.fan_out_width` — unchanged key, but the 64 ceiling is gone; it now validates only as a nonnegative bound and acts as the default `max_fan_out` seed.
- No other new keys. Strategy, routing, and time travel carry no configuration — they are authored (strategy/routes) or invoked (rerun/fork) per run.

## Native Tools

All in toolset `compozy__loops`, `additionalProperties:false`, property names matching CLI flags.

| Tool | Mode | Purpose |
| --- | --- | --- |
| `compozy__loop_requests` | read | List human requests (`workspace?`, `run_id?`, `state?`, `cursor?`, `limit?`) |
| `compozy__loop_request` | read | One request in full (`run_id`, `node_id`, `item_index?`) — full redacted context + schemas |
| `compozy__loop_respond` | mutating · gated `loops.respond` | Answer/decide (`run_id`, `node_id`, `decision?`, `payload?`, `note?`, `item_index?`) |
| `compozy__loop_node_amend` | mutating · gated `loops.respond` | Amend parked output (`run_id`, `node_id`, `payload`, `item_index?`, `reason?`) |
| `compozy__loop_diff` | read | Diff (`run_id`, `generation?`, `against_generation?`, `against_run?`) |
| `compozy__loop_rerun` | mutating · gated `loops.timetravel` | Rerun (`run_id`, `from_node`, `item_index?`, `reason?`, `request_id?`) |
| `compozy__loop_fork` | mutating · gated `loops.timetravel` | Fork (`run_id`, `generation`, `inputs?`, `reason?`, `request_id?`) |

Example call and result:

```json
// compozy__loop_respond
{ "run_id": "lr_01j9...", "node_id": "pick_envs", "payload": { "environments": ["dev", "staging"] } }
```

```json
{ "ok": true, "run_id": "lr_01j9...", "node_id": "pick_envs", "decision": "respond", "state": "answered" }
```

Self-response is denied for the starting agent regardless of capability (`respond_self_denied`), exactly like `loops.approve` today.

## Errors

| Condition | Code | Where | Points to |
| --- | --- | --- | --- |
| Answer/edit/amend payload fails the declared shape | `request_validation_failed` + field details | respond/amend, all surfaces | Fix the payload; the expected shape is in the request payload (`expect`) |
| Request already answered | `request_already_answered` | respond | Read the recorded decision in the response body |
| Request expired | `request_expired` | respond | The expiry route already ran; see run history |
| Request canceled with its node/run/strategy | `request_canceled` | respond | Nothing to answer; run history names the canceling cause |
| Agent responder without node opt-in | `respond_not_permitted` | respond | Author must set `responders.agents: allow` on the node |
| Starting agent answers its own run | `respond_self_denied` | respond/approve | Route to a human or another permitted agent |
| Amend on a running node | `amend_not_parked` | amend | Pause the node first (`loop node pause`) |
| Amend on a cell with no output yet | `amend_no_output` | amend | Only settled outputs are amendable; answer/resume the cell instead |
| Amend without a declared output shape | `amend_schema_missing` | amend | Declare `produces`/`output_schema` on the node |
| `--request-id` reused with different arguments | `timetravel_key_reuse` | rerun/fork | Use a new request id (or omit it for digest-keyed idempotency) |
| Rerun target not settled | `rerun_node_unsettled` | rerun | Use node resume/requeue verbs for parked lanes |
| Rerun while a generation is in flight | `rerun_busy` | rerun | Wait for the generation to settle, or pause first |
| Fork from an unknown generation | `fork_generation_unknown` | fork | `loop status` lists valid generations |
| Agent rerun/fork of its own executing run | `timetravel_self_denied` | rerun/fork | Wait for terminal state, or have an operator do it |
| Diff across different loops | `diff_cross_loop` | diff | Diff compares runs of the same loop only |
| Router without `default` | lint `route_default_missing` | create/validate | Add `default:` to the router |
| Route target not a forward destination | lint `route_target_invalid` | create/validate | Routes must point at declared forward nodes |
| `best_effort` without `missing: acceptable` | lint `strategy_coverage_undeclared` | create/validate | Declare that missing branches are acceptable |
| `strategy` threshold invalid (0%, >100%) | lint `strategy_threshold_invalid` | create/validate | Use 1–100% or a positive count |
| `bind_as`/`index_as` collision | lint `iteration_name_conflict` | create/validate | Pick names unique in the nesting chain and not reserved |
| `no_progress.hash_fields` present | lint `unknown_parameter` | create/validate | The field was removed; delete it from the definition |
| `on_result` uses the removed `branch` action string | lint `route_action_removed` | create/validate | Use the object form `{ route: <node_id> }` |
| Broken routing condition at runtime | node failure class `authoring`, code `predicate_evaluation_failed` | run history | Route or absorb via `on_error`, or fix the condition |
| Broken `stop_when` at runtime | run exits with diagnostic `predicate_evaluation_failed` | run history | Fix the condition; the run does not wedge |
```
