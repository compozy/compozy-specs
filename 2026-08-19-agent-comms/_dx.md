# Developer Experience: Agent Comms — Typed Calls, Mailbox, and Subagents

Public-surface contract for agent-comms. Companion to `_spec.md` (Part II serves
this surface) and `_tests.md` (E2E journeys use these exact invocations).

Vocabulary: a **call** delegates work to an agent and carries a **result contract**
(`expect`). A finished child is **parked** — still addressable until its TTL. This is
never a "handoff": the caller always keeps control.

## Golden Path

Thirty seconds from nothing to a typed result.

```bash
# 1. Author a specialist (workspace agent definition)
$ cat .compozy/agents/reviewer/AGENT.md
---
name: reviewer
description: Reviews a diff and returns structured findings with severity.
provider: claude
model: claude-opus-5
---
You are a strict code reviewer. Review what you are given and report findings.

# 2. Any session in this workspace can now call it — here, from the CLI
$ compozy call reviewer "Review the diff in HEAD~1..HEAD of this repo" \
    --expect '{"findings":[{"file":"","line":0,"severity":"","note":""}],"verdict":""}'
call        call_01JBD8G2K7Q9
agent       reviewer
child       ses_01JBD8G2MZTX
state       running
idle_ttl    1h (clock runs only while the child is parked)

# 3. Await the typed result (bounded; resumable)
$ compozy call await call_01JBD8G2K7Q9 --timeout 120s
state       completed
verdict     returned            # returned | extracted | repaired
result      {"findings":[{"file":"internal/loop/action.go","line":88,"severity":"warning",
            "note":"error swallowed on retry path"}],"verdict":"needs-changes"}
```

Inside an agent session the same journey is two native tools: `compozy__agent_call`
to delegate, and the completion arrives on its own as a wake carrying the result —
no polling. The operator watches the whole exchange live in the **Agents** app.

## YAML

### AGENT.md — the `description` field

Every agent definition may now carry a `description`. It is what other agents read
when choosing a specialist: the call surface embeds the roster (name + description)
of every definition visible in the workspace (workspace shadows global).

```yaml
---
name: reviewer
description: Reviews a diff and returns structured findings with severity.
provider: claude
model: claude-opus-5
tools: [read, grep]
---
You are a strict code reviewer...
```

Definitions without `description` stay valid; they appear in the roster with an
empty description. The authoring maximum is 500 characters — over it, loading
fails with the bound named (roster injection additionally truncates its display
at 120 characters). File lives at `.compozy/agents/<name>/AGENT.md` (workspace) or
`$COMPOZY_HOME/agents/<name>/AGENT.md` (global); loaded exactly as today.

### Tasks — result contracts

A task may declare what a valid worker result looks like. Same contract regime,
same repair semantics (one resubmission round), enforced at completion admission:

```bash
$ compozy task create "Fix the loop retry bug" --expect @fix-contract.json
task        task_01JBD9AAAA
expect      sha256:7c1a…            # digest + budget echoed on every read surface

$ compozy task update task_01JBD9AAAA --expect @fix-contract-v2.json
task        task_01JBD9AAAA
expect      sha256:44d0…            # applies to FUTURE runs; in-flight runs keep their start-time contract
```

HTTP/UDS carry the same optional field: `POST/PATCH …/tasks` accept
`"expect": {…}` (+ optional `"result_budget"`, `"result_overflow"`), and task
read payloads return `expect_digest` + the effective budget. The native task
create/update tools accept the same `expect` argument. A worker completing a
contracted run with a non-conforming result gets the sanitized validator errors
back as a typed completion rejection and may resubmit once; the second failure
records the run's typed invalid-result outcome.

### Result contracts — two accepted forms

`expect` (and Loop `output_schema`) accepts either a **full JSON Schema** or the
**example-shape shorthand** used in the golden path: a plain object whose keys
become required properties and whose values illustrate the types. Both
normalize to the same canonical contract — the shorthand below pins the same
digest as its expanded schema:

```json
{"findings": [{"file": "", "line": 0, "severity": "", "note": ""}], "verdict": ""}
```

```json
{"type": "object", "required": ["findings", "verdict"], "properties": {
  "findings": {"type": "array", "items": {"type": "object",
    "required": ["file", "line", "severity", "note"],
    "properties": {"file": {"type": "string"}, "line": {"type": "number"},
                   "severity": {"type": "string"}, "note": {"type": "string"}}}},
  "verdict": {"type": "string"}}}
```

Anything that is neither form → `call_expect_invalid` with the schema error.

### Loops

Loop DSL is unchanged. A `run-agent` node's `output_schema` and a call's `expect`
are the same kind of result contract and behave identically: validated when the
result is produced **and** when it settles; an invalid payload can never settle as
succeeded. What loops learned, calls know — and vice versa.

## CLI

The `compozy spawn` verb no longer exists — `compozy call` replaces it (a call that
only creates is just a call whose work is the first prompt). All verbs support
`-o human|json|jsonl|toon`.

### `compozy call <agent|session-id> <prompt>`

```bash
$ compozy call reviewer "Review HEAD~1..HEAD" --expect @review-contract.json \
    --idle-ttl 1h --idempotency-key rev-2026-08-20-01
call        call_01JBD8G2K7Q9
agent       reviewer
child       ses_01JBD8G2MZTX
state       running
idle_ttl    1h (clock runs only while the child is parked)
```

```bash
$ compozy call reviewer "..." --idempotency-key rev-2026-08-20-01   # replay
call        call_01JBD8G2K7Q9
state       completed
replayed    true
```

`--expect` takes inline JSON or `@file` (either contract form above). Optional:
`--strict` (disable prose extraction for this call — a contracted child that
never invokes the return act settles `completed-without-result` directly),
`--result-budget 512KiB` / `--result-overflow store|reject` (per-call override
of `[calls.results]`, capped by `max_budget` and snapshotted immutably on the
call), `--idle-ttl` (parked-idle
ceiling for the child; the clock runs only while it is parked and suspends while
a call is in flight — a working child is never reaped by the clock), `--deadline`
(opt-in call timeout; **there is no default** — a call runs until it settles, is
canceled, or the parent drains; stalls are handled by session activity
supervision, not wall-clock), `--runtime` (`provider/model/reasoning/speed`
override), and the four permission-narrowing flags `--tools`, `--skills`,
`--workspace-paths`, `--network-channels` (subset-only against your own set).
Exit code `0` on acceptance.

Follow-up uses the same verb with the child's session id:

```bash
$ compozy call ses_01JBD8G2MZTX "One more thing: check the tests too"
call        call_01JBD8H9PW2M
child       ses_01JBD8G2MZTX      # same child, context preserved (revived if parked)
state       running
```

Failure examples:

```bash
$ compozy call reviewre "..."
error: call_agent_unknown: no agent definition named "reviewre"
available: reviewer — Reviews a diff and returns structured findings with severity.
           scout — Maps a codebase area and returns entry points.
try: compozy call reviewer "..."
# exit code 2

$ compozy call ses_01JBD8G2MZTX "..."       # after TTL
error: call_target_expired: session ses_01JBD8G2MZTX expired 2026-08-20T19:12:04Z
try: compozy call reviewer "..." to start a fresh child
# exit code 2
```

### Call states

A call is always in exactly one of nine states. `queued` — admitted, activation
or delivery pending (visible, durable). `running` — the child is working.
Terminals: `completed` (typed result recorded), `invalid-result` (contract
failed after the repair round), `completed-without-result` (contracted child
finished with nothing to admit), `failed` (spawn/runtime/contract-fault reason
attached), `canceled` (caller/operator), `timeout` (opt-in deadline),
`expired` (target idle past its TTL). `--state` filters accept any
comma-separated subset of these nine.

### `compozy call list` / `compozy call show <call-id>`

```bash
$ compozy call list --state running,completed --caller ses_01JBD7ZZAAAA --limit 3
CALL              AGENT      CHILD             STATE      AGE    RESULT
call_01JBD8G2K7Q9 reviewer   ses_01JBD8G2MZTX  completed  2m     returned (312 B)
call_01JBD8H9PW2M reviewer   ses_01JBD8G2MZTX  running    40s    —
call_01JBD8J1XKCV scout      ses_01JBD8J1ZQ8F  running    12s    —
```

```bash
$ compozy call show call_01JBD8G2K7Q9
call        call_01JBD8G2K7Q9
agent       reviewer
caller      ses_01JBD7ZZAAAA (session)
child       ses_01JBD8G2MZTX
state       completed
verdict     returned
contract    sha256:9f2c…
result      {"verdict":"needs-changes"} (312 B — full payload: compozy call result)
settled     2026-08-20T18:14:11Z (2m07s)

$ compozy call show call_01JBD8G2K7Q9 -o json
{
  "call_id": "call_01JBD8G2K7Q9",
  "workspace_id": "ws_main",
  "caller": {"kind": "session", "id": "ses_01JBD7ZZAAAA"},
  "agent": "reviewer",
  "child_session_id": "ses_01JBD8G2MZTX",
  "state": "completed",
  "verdict": "returned",
  "expect_digest": "sha256:9f2c…",
  "result_preview": {"verdict": "needs-changes"},
  "result_bytes": 312,
  "depth": 1,
  "created_at": "2026-08-20T18:12:04Z",
  "settled_at": "2026-08-20T18:14:11Z",
  "provenance": {"produced_by": "reviewer", "admitted": "returned"}
}
```

### `compozy call await <call-id>` / `compozy call cancel <call-id>` / `compozy call result <call-id>`

```bash
$ compozy call await call_01JBD8H9PW2M --timeout 300s
state       timeout
resume      cawait_01JBD8KQ33
try: compozy call await call_01JBD8H9PW2M --resume cawait_01JBD8KQ33
# exit code 3 (timeout is a checkpoint, not a failure)

$ compozy call cancel call_01JBD8H9PW2M --reason "superseded by rev-02"
state       canceled

$ compozy call result call_01JBD8G2K7Q9 -o json      # the FULL stored payload
{"findings":[{"file":"internal/loop/action.go","line":88,"severity":"warning",
"note":"error swallowed on retry path"}],"verdict":"needs-changes"}
```

`--timeout` clamps to a 30-minute maximum (over-max values are clamped, and the
output says so).

### `compozy call publish <call-id>` / `compozy session stop <id> --subtree`

```bash
$ compozy call publish call_01JBD8G2K7Q9 --channel eng-room --thread thread_reviews
published   true
message     msg_9f2c…

$ compozy call publish call_01JBD8H9PW2M --channel eng-room
error: call_publish_not_settled: call is running — only completed calls publish
try: compozy call await call_01JBD8H9PW2M
# exit code 2

$ compozy session stop ses_01JBD7ZZAAAA --subtree --reason "superseded"
stopped     ses_01JBD7ZZAAAA
drained     3 children, 2 open calls closed (parent-terminal), 1 completed result preserved
```

### `compozy message send <session-id> <text>` / `compozy message list`

```bash
$ compozy message send ses_01JBD8G2MZTX "Prioritize the loop package first"
message     msg_01JBD8M2R4V7
delivery    queued              # delivered-into-turn | woke | queued | failed

$ compozy message list --session ses_01JBD8G2MZTX --limit 2
MESSAGE           FROM                TO                 DELIVERY             AGE
msg_01JBD8M2R4V7  operator            ses_01JBD8G2MZTX   delivered-into-turn  40s
msg_01JBD8KX9QQ1  ses_01JBD8G2MZTX    ses_01JBD7ZZAAAA   woke                 44s

$ compozy message list --session ses_01JBD8G2MZTX --limit 2 -o jsonl
{"message_id":"msg_01JBD8M2R4V7","from":{"kind":"operator"},"to":"ses_01JBD8G2MZTX","text":"Prioritize the loop package first","delivery":"delivered-into-turn","created_at":"2026-08-20T18:13:02Z"}
{"message_id":"msg_01JBD8KX9QQ1","from":{"kind":"session","id":"ses_01JBD8G2MZTX"},"to":"ses_01JBD7ZZAAAA","text":"Blocked: repo has no tests for internal/loop — proceed reviewing anyway?","delivery":"woke","created_at":"2026-08-20T18:12:58Z"}
```

### `compozy agent list`

```bash
$ compozy agent list -o json
[{"name": "reviewer", "description": "Reviews a diff and returns structured findings with severity.",
  "scope": "workspace", "shadowed": false, "digest": "sha256:2b8e…"},
 {"name": "scout", "description": "Maps a codebase area and returns entry points.",
  "scope": "global", "shadowed": false, "digest": "sha256:91aa…"}]
```

Same fields on `GET /api/workspaces/{workspace_id}/agents` and in the
`compozy__agent_list` tool result — exactly what roster injection renders,
shadow-aware (`"shadowed": true` rows are inactive name collisions).

## HTTP / UDS API

All routes are workspace-scoped and exist on HTTP and UDS identically.

### `POST /api/workspaces/{workspace_id}/calls`

```json
{
  "target": {"agent": "reviewer"},
  "prompt": "Review HEAD~1..HEAD",
  "expect": {"findings": [{"file": "", "line": 0, "severity": "", "note": ""}], "verdict": ""},
  "idle_ttl_seconds": 3600,
  "deadline_seconds": 300,
  "strict": false,
  "result_budget": "512KiB",
  "result_overflow": "store",
  "idempotency_key": "rev-2026-08-20-01"
}
```

(`strict`, `result_budget`, `result_overflow` are optional on every create
surface — tool, HTTP/UDS, and batch items — with the same semantics as the CLI
flags; budget and deadline participate in idempotency identity, so replaying a
key with different values is `call_idempotency_conflict`.)

(`deadline_seconds` is optional on every create surface — tool, HTTP/UDS, batch
items — always a positive integer of seconds; invalid values → `422
call_deadline_invalid`.)

`201` →

```json
{
  "call_id": "call_01JBD8G2K7Q9",
  "child_session_id": "ses_01JBD8G2MZTX",
  "state": "running",
  "replayed": false,
  "idle_expires_at": null
}
```

(`idle_expires_at` is `null` while a call is in flight; it becomes a timestamp
when the child parks, and resets on the next contact.)

Batch: the same route accepts `{"tasks": [{...}, {...}]}` → status `200` with a
per-item array, each entry `{call_id, state}` or `{error}`. Follow-up: `"target":
{"session_id": "ses_01JBD8G2MZTX"}`.

Failures: `404 call_agent_unknown` (body carries `available` roster) · `409
call_idempotency_conflict` · `410 call_target_expired` · `422
call_expect_invalid` / `call_widening_rejected` / `call_batch_over_cap` · `403
call_workspace_denied`.

### `GET /api/workspaces/{workspace_id}/calls?state=…&caller=…&cursor=…`

`200` → `{"items": [<call summary>…], "next_cursor": "…"}` — same summary shape as
`compozy call show`.

### `GET /api/workspaces/{workspace_id}/calls/{call_id}` · `/result` · `POST …/cancel` · `POST …/await` · `POST …/publish`

- `GET …/{call_id}` → the full call record (`200`; `404`).
- `GET …/{call_id}/result` → the stored payload, whole (`200`; `409
  call_not_settled`; `404`).
- `POST …/{call_id}/cancel` `{"reason": "…"}` → `200 {"state": "canceled"}`;
  idempotent (`200` with current terminal state).
- `POST …/{call_id}/await` `{"timeout_ms": 300000, "resume": "cawait_…?"}` →
  `200 {"state": "completed", "verdict": "returned", "result_preview": {…}}` or
  `200 {"state": "timeout", "resume": "cawait_01JBD8KQ33"}` (`timeout_ms` clamps
  at 30 minutes).
- `POST …/{call_id}/publish` `{"channel": "eng-room", "thread_id": "thread_reviews"}` →
  `200 {"network_message_id": "msg_9f2c…", "published": true}`; `422
  call_publish_no_participation` · `409 call_publish_not_settled`. Channel-thread
  conversations only — there is no direct-room publish target.

### `POST /api/workspaces/{workspace_id}/sessions/{id}/stop` gains `subtree`

`{"subtree": true, "reason": "superseded"}` → `200 {"stopped_children": 3,
"closed_calls": 2, "preserved_results": 1}` — the same drain the CLI flag and
the Agents app control invoke; idempotent on repeat.

### `POST /api/workspaces/{workspace_id}/messages` · `GET …/messages?session=…`

```json
{"to": {"session_id": "ses_01JBD8G2MZTX"}, "text": "Prioritize the loop package first"}
```

`202` → `{"message_id": "msg_01JBD8M2R4V7", "delivery": "queued"}`. Failures: `429
message_rate_limited` · `413 message_too_large` · `409 message_target_blocked`
(target awaits a human decision) · `410 call_target_expired` · `403
call_workspace_denied`.

## config.toml

```toml
[calls]
max_depth = 3                 # delegation nesting wall; at the wall the call tool is absent
max_batch = 8                 # tasks per call invocation; over → call_batch_over_cap (whole batch)
max_children = 5              # per-parent admission wall: a call over this cap REJECTS (typed error)
max_active_per_root = 32      # execution budget per governed tree; admitted work over it QUEUES (visible)
idle_ttl = "1h"               # parked-idle ceiling; the clock runs only while the child is parked

[calls.results]
default_budget = "256KiB"     # per-contract result ceiling (contract may override, capped by max_budget)
max_budget = "4MiB"
overflow = "store"            # store: keep whole payload, project bounded previews | reject

[calls.messages]
rate_limit_per_minute = 30    # per sender; over → message_rate_limited
dedup_window = "30s"          # identical repeats inside the window are dropped
pending_cap = 50              # queued-undelivered (transport backlog) ceiling per recipient - no read/seen semantics
max_bytes = "64KiB"           # per message; over → message_too_large
```

Queued-message policy: a message queued for a parked or busy child stays
durably queued and is delivered at the next boundary or revival; when the
target expires (idle TTL) or is drained, every queued message terminalizes
`failed` with that reason — observable on the message record, never dropped
silently. There is no additional retry knob.

Every key is agent-manageable:

```bash
$ compozy config get calls.max_depth
3
$ compozy config set calls.max_batch 4
calls.max_batch = 4
``` Observable effects: `max_depth` changes apply to
new spawns; `idle_ttl` applies at call time and suspends while a call is in
flight — a working child is never clock-reaped; there is no default deadline
(`--deadline` is per-call opt-in; stall handling belongs to session activity
supervision); `overflow = "reject"` turns over-budget results into
`call_result_over_budget` failures.

## Native Tools

### `compozy__agent_call`

```json
{
  "agent": "reviewer",
  "prompt": "Review HEAD~1..HEAD and report findings.",
  "expect": {"findings": [{"file": "", "line": 0, "severity": "", "note": ""}], "verdict": ""},
  "idempotency_key": "rev-01"
}
```

→

```json
{"call_id": "call_01JBD8G2K7Q9", "child_session_id": "ses_01JBD8G2MZTX",
 "state": "running", "replayed": false, "idle_expires_at": null}
```

The tool's `agent` parameter description carries the live roster: `reviewer —
Reviews a diff…`, `scout — Maps a codebase area…` (workspace shadows global).
Follow-up: `{"session_id": "ses_01JBD8G2MZTX", "prompt": "…"}`. Batch:
`{"tasks": [{"agent": "scout", "prompt": "…"}, {"agent": "reviewer", "prompt": "…",
"expect": {…}}]}` → per-item results. At the depth wall this tool is absent from
the child's toolset, and every child's context states its literal remaining depth.

### `compozy__call_return` — the child's terminal act

```json
{"result": {"findings": [{"file": "internal/loop/action.go", "line": 88,
 "severity": "warning", "note": "error swallowed on retry path"}],
 "verdict": "needs-changes"}}
```

→ `{"call_id": "call_01JBD8G2K7Q9", "state": "completed"}` — validated against the
call's contract at admission; recording the result and settling the call are one
act. Secret-shaped values are hash-redacted **before** validation, so everything
downstream — including error text — is already sanitized. On contract violation
the tool returns the validator's errors verbatim (from the sanitized payload) and
the child gets exactly one repair attempt; a second failure settles the call
`invalid-result`. A single-key wrapper around a valid payload is unwrapped, not
failed. Infrastructure failures never consume the repair attempt.

### `compozy__call_await` · `compozy__call_cancel` · `compozy__call_result`

```json
{"call_ids": ["call_01JBD8G2K7Q9", "call_01JBD8J1XKCV"], "timeout_ms": 120000}
```

→ `{"settled": [{"call_id": "call_01JBD8G2K7Q9", "state": "completed",
"verdict": "returned", "result_preview": {…}}], "pending": ["call_01JBD8J1XKCV"],
"outcome": "partial", "resume": "cawait_01JBD8KQ33"}`

`compozy__call_cancel` `{"call_id": "…", "reason": "…"}` → `{"state": "canceled"}`.
`compozy__call_result` `{"call_id": "…"}` → the whole stored payload.
`timeout_ms` clamps to the 30-minute maximum on every surface — over-max requests
are clamped (never rejected) and the response reflects the clamped value.

### `compozy__call_publish` — publish a settled call into a Network conversation

```json
{"call_id": "call_01JBD8G2K7Q9", "channel": "eng-room", "thread_id": "thread_reviews"}
```

→ `{"network_message_id": "msg_9f2c…", "published": true}` — one-way only: the
evidence (bounded result preview + call identity + fetch reference) becomes a
`say` with `intent:"result"` in that conversation, attributed to you, under
Network's own rules. Publishing the same call to the same conversation again
returns the recorded message id with `"published": false` (replay, not a second
post); a different conversation publishes anew. Nothing ever flows back from
Network into a call. Requires
your active Live participation; the call must be `completed` with a valid result — no other state (running or resultless terminals) has evidence to publish, and all of them reject with `call_publish_not_settled`.

### `compozy__session_stop` gains `subtree`

```json
{"session_id": "ses_01JBD7ZZAAAA", "subtree": true, "reason": "superseded"}
```

→ stops the session **and drains its whole delegation subtree** (depth ≤ 3, cycle-safe):
children stop, open calls close with the parent-terminal reason, completed
results stay fetchable. This is the same operation the Agents app "stop subtree"
control invokes.

### `compozy__agent_message`

```json
{"to": "parent", "text": "Blocked: repo has no tests for internal/loop — proceed reviewing anyway?"}
```

→ `{"message_id": "msg_01JBD8KX9QQ1", "delivery": "woke"}`

`to` is `"parent"` or a session id inside the lineage/grant. Messages are text,
bounded, durable-first; delivery is never mid-tool — boundaries only, new turn
when idle. Every delivered message renders to its recipient provenance-stamped
("from agent reviewer (ses_…), not the operator"), inside a bounded untrusted
frame; embedded commands arrive inert; nothing in a message can approve a pending
permission.

### The completion wake, as the parent sees it

When a child settles a call, the parent's next turn opens with:

```
Call completed: reviewer (call_01JBD8G2K7Q9) → completed.
Result preview: {"verdict":"needs-changes"} (312 B, contract sha256:9f2c…)
Fetch the full result with compozy__call_result.
```

carrying the same fields as structured metadata. Resultless terminals wake with
the state and the typed reason instead — exactly:

```
Call failed: reviewer (call_01JBD8G2K7Q9) → invalid-result.
Reason: result did not satisfy the contract after 1 repair attempt (2 issues).
Inspect with compozy__call_result (attempts and errors are recorded).
```

The wake and the outcome are one delivery in every case.

## Errors

| Condition | Code | Surfaces | Points to |
| --- | --- | --- | --- |
| Unknown agent name | `call_agent_unknown` | tool, CLI (exit 2), HTTP 404 | the current roster, inline |
| Empty prompt | `call_prompt_empty` | tool, CLI (exit 2), HTTP 422 | a call always carries work |
| Empty batch | `call_batch_empty` | tool, CLI (exit 2), HTTP 422 | at least one task |
| Per-parent children cap | `call_children_cap` | tool, CLI (exit 2), HTTP 422 | the cap and current count (admission wall — rejects, never queues) |
| Target outside lineage/grant | `call_target_denied` | tool, CLI (exit 2), HTTP 403 | your own children or a grant |
| Target expired (idle past its TTL) | `call_target_expired` | tool, CLI (exit 2), HTTP 410 | calling the agent fresh |
| Target never existed | `call_target_not_found` | tool, CLI (exit 2), HTTP 404 | `compozy call list` |
| Depth wall reached | `call_depth_exceeded` | tool, HTTP 422 | the configured `calls.max_depth` |
| Batch over cap | `call_batch_over_cap` | tool, CLI (exit 2), HTTP 422 | `calls.max_batch`; nothing partial ran |
| Permission widening | `call_widening_rejected` | tool, CLI (exit 2), HTTP 422 | the widening atoms, named |
| Idempotency conflict | `call_idempotency_conflict` | tool, CLI (exit 2), HTTP 409 | the original call id |
| Invalid result (after repair) | terminal state `invalid-result` | call record, wake | both attempts' errors |
| Result over budget (`overflow = "reject"`) | `call_result_over_budget` | return tool | the declared budget |
| Contract malformed | `call_expect_invalid` | tool, CLI (exit 2), HTTP 422 | the schema error, verbatim |
| Deadline invalid (zero/negative/non-integer) | `call_deadline_invalid` | tool, CLI (exit 2), HTTP 422 | the accepted form: positive integer seconds |
| Return with no bound call | `call_return_unbound` | return tool | the child's active calls |
| Call already settled (return) | `call_already_settled` | return tool, HTTP 409 | the terminal state — cancel on a terminal call is an idempotent 200 no-op, never this error |
| Message rate limit | `message_rate_limited` | tool, HTTP 429 | the window and reset |
| Identical repeat in dedup window | `message_duplicate` | tool, HTTP 409 | the window; the original message id |
| Unknown message id | `message_not_found` | tool, CLI (exit 2), HTTP 404 | `compozy message list` |
| Result fetched before settle | `call_not_settled` | result tool/route, HTTP 409 | `compozy call await` |
| Message too large | `message_too_large` | tool, CLI (exit 2), HTTP 413 | `calls.messages.max_bytes` |
| Target awaiting human decision | `message_target_blocked` | tool, HTTP 409 | the decision surface, not messaging |
| Cross-workspace target | `call_workspace_denied` | every surface, HTTP 403 | nothing — hard denial |
| Publish without Network participation | `call_publish_no_participation` | publish tool, CLI (exit 2), HTTP 422 | joining a channel first |
| Publish a call that is not `completed`-with-result (running or any resultless terminal) | `call_publish_not_settled` | publish tool, CLI (exit 2), HTTP 409 | awaiting the call first |

Every error is deterministic: same condition, same code, same shape, every surface.
