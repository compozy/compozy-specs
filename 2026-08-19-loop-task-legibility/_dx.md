# Developer Experience: Loop & Task Legibility

Public-surface contract for the loop/task legibility program. Companion to `_spec.md` (Part II serves this surface) and `_tests.md` (E2E journeys use these exact invocations). Web surfaces live in `_uiux.md`.

## Golden Path

Supervise a loop run end-to-end without the web app, while your task list stays yours:

```console
$ compozy loop start revisao-paralela --input tema="rate limiting"
Run looprun-8f3ab2c1d4e5f607 started (revisao-paralela, round 1).

$ compozy task list
ID            STATUS       TITLE
tsk-92ab41    in_progress  Escrever post sobre memoria
tsk-88cf02    ready        Revisar landing page
# zero loop rows — loop execution records are excluded by default

$ compozy loop why looprun-8f3ab2c1d4e5f607
RUNNING · round 1 — implementar is running (2m14s)
Nothing needs you. 1 of 6 steps done.
Watch: compozy loop events looprun-8f3ab2c1d4e5f607 --follow

$ compozy loop events looprun-8f3ab2c1d4e5f607 --follow
[g1] step implementar started (session ses-77120a3f)
[g1] step implementar succeeded (4m02s, attempt 1)
[g1] steps revisor-seguranca, revisor-perf, revisor-estilo started (fan-out 3)
[g1] step revisor-perf needs you: approve gate "aplicar-correcoes"
^C

$ compozy loop why looprun-8f3ab2c1d4e5f607
NEEDS YOU · round 1 — approval "aplicar-correcoes" waiting 3m
The gate is asking whether to apply the reviewers' corrections.
Unblock: compozy loop approve looprun-8f3ab2c1d4e5f607 --gate aplicar-correcoes
```

## CLI

### `compozy task list` — calm default + explicit include

Default excludes loop execution records (coordinator + cells) on every scope:

```console
$ compozy task list -o json
{"items":[{"id":"tsk-92ab41","title":"Escrever post sobre memoria","status":"in_progress", ...}],
 "next_cursor":""}
```

Explicit include (typed flag, never inferred from other filters):

```console
$ compozy task list --include-loop
ID                                            STATUS     LOOP                      TITLE
loop.looprun-8f3ab2c1d4e5f607.coordinator     ready      revisao-paralela · run    Loop run revisao-paralela
loop.looprun-8f3ab2c1d4e5f607.g1.node.implementar.0
                                              completed  revisao-paralela · g1     step implementar
tsk-92ab41                                    in_progress —                        Escrever post sobre memoria
```

```console
$ compozy task list --include-loop --loop-run looprun-8f3ab2c1d4e5f607 -o json
{"items":[
  {"id":"loop.looprun-8f3ab2c1d4e5f607.coordinator","status":"ready",
   "loop":{"run_id":"looprun-8f3ab2c1d4e5f607","loop_name":"revisao-paralela","role":"coordinator"}},
  {"id":"loop.looprun-8f3ab2c1d4e5f607.g1.node.implementar.0","status":"completed",
   "loop":{"run_id":"looprun-8f3ab2c1d4e5f607","loop_name":"revisao-paralela","role":"cell",
           "generation":1,"node_id":"implementar","item_index":0}}
], "next_cursor":""}
```

- `--loop-run <id>` implies `--include-loop` and scopes to that run's records.
- `--parent <task-id>` keeps returning that parent's children (explicit parentage is already an explicit request) — no flag needed.
- Exit code 0; unknown `--loop-run` returns an empty list (it is a filter, not a lookup).

### `compozy loop runs` — which runs need me, server-ordered

The runs list is server-owned end to end: needs-you runs order first, then active, then terminal — **before pagination**, so the order holds across pages. Each row carries a server-computed attention summary and step/round progress (this is MVP operational state, not the deferred cross-run analytics):

```console
$ compozy loop runs
STATUS      LOOP               PROGRESS         STARTED  DURATION
NEEDS YOU   revisao-paralela   step 4/6 · r1    18:32    22m   approval "aplicar-correcoes" (3m)
running     fabrica-assistida  step 2/9 · r1    18:41    13m
done        revisao-paralela   —                17:40    18m
```

```console
$ compozy loop runs -o json
{"items":[
  {"run_id":"looprun-8f3ab2c1d4e5f607","loop_name":"revisao-paralela","status":"running",
   "attention":{"kind":"approval","count":1,"since":"2026-08-19T18:44:01Z"},
   "progress":{"round":1,"steps_done":4,"steps_total":6}, ...}
], "next_cursor":""}
```

`attention` is absent when nothing waits on you; existing filters (`--loop`, `--status`, …) unchanged.

### `compozy loop nodes` — run roster (healthy nodes included)

Existing workspace inventory keeps its exception states. Scoped to one run, `--all` returns the complete node × round roster:

```console
$ compozy loop nodes --run looprun-8f3ab2c1d4e5f607 --all
ROUND  STEP              STATE      ATTEMPT  DURATION  SESSION
g1     implementar       succeeded  1        4m02s     ses-77120a3f
g1     revisor-seguranca succeeded  1        1m48s     ses-9a02bb17
g1     revisor-perf      waiting    1        —         ses-c3f00e42   approval "aplicar-correcoes"
g1     revisor-estilo    succeeded  2        2m31s     ses-5d871c99   recovered on attempt 2
g1     sintetizador      queued     —        —         —
g1     saida             pending    —        —         —
```

```console
$ compozy loop nodes --run looprun-8f3ab2c1d4e5f607 --all -o json
{"run_id":"looprun-8f3ab2c1d4e5f607","loop_name":"revisao-paralela",
 "nodes":[
   {"generation":1,"node_id":"revisor-estilo","item_index":0,"state":"succeeded",
    "attempt":2,"started_at":"2026-08-19T18:41:07Z","ended_at":"2026-08-19T18:43:38Z",
    "session_id":"ses-5d871c99","cell_task_id":"loop.looprun-8f3ab2c1d4e5f607.g1.node.revisor-estilo.0",
    "attempts":[
      {"attempt":1,"state":"failed","failure_class":"tool_error","ended_at":"2026-08-19T18:40:52Z"},
      {"attempt":2,"state":"succeeded","ended_at":"2026-08-19T18:43:38Z"}]},
   {"generation":1,"node_id":"sintetizador","item_index":0,"state":"queued","attempt":0,"attempts":[]}
 ],
 "fanout_rollups":[{"generation":1,"node_id":"revisores","done":2,"total":3,"failed":0}],
 "next_cursor":""}
```

- `--state` gains `running`, `queued`, `succeeded`, `all` when `--run` is set (workspace scope keeps the four exception states).
- `--generation N` filters one round. Fan-out items page under `next_cursor`; rollups are always present so agents need not fetch every item.

### `compozy loop why` — the briefing

Same verdict the run page leads with, as one plain sentence plus the concrete unblock:

```console
$ compozy loop why looprun-8f3ab2c1d4e5f607 -o json
{"run_id":"looprun-8f3ab2c1d4e5f607","status":"running","tone":"needs_you",
 "headline":"Approval \"aplicar-correcoes\" waiting 3m",
 "detail":"The gate is asking whether to apply the reviewers' corrections. 4 of 6 steps done in round 1.",
 "blockers":[{"kind":"approval","gate_id":"aplicar-correcoes","waiting_since":"2026-08-19T18:44:01Z",
              "unblocker":"compozy loop approve looprun-8f3ab2c1d4e5f607 --gate aplicar-correcoes"}],
 "progress":{"round":1,"steps_done":4,"steps_total":6},
 "usage":{"tokens":82400,"cost_usd":0.31,"budget_used_pct":12,"duration":"9m40s"}}
```

Healthy and terminal runs answer too — never an empty object. Terminal briefings carry the outcome and produced artifacts as **typed fields** (the human text is a projection of them):

```console
$ compozy loop why looprun-77aa01b2c3d4e5f6
DONE · finished 2026-08-19 17:58 after 2 rounds (18m12s)
Spent 214k tokens · $0.87 · 38% of budget
Produced: post-final.md (output "saida")

$ compozy loop why looprun-77aa01b2c3d4e5f6 -o json
{"run_id":"looprun-77aa01b2c3d4e5f6","status":"done","tone":"ok",
 "headline":"Finished after 2 rounds (18m12s)","detail":"…","blockers":[],
 "outcome":{"status":"done","cause":"verified","at":"2026-08-19T17:58:12Z"},
 "artifacts":[{"name":"post-final.md","output":"saida","availability":"available","ref":"blob-2f81…"}],
 "progress":{"round":2,"steps_done":6,"steps_total":6},
 "usage":{"tokens":214500,"cost_usd":0.87,"budget_used_pct":38,"duration":"18m12s"}}
```

`outcome.actor_kind`/`actor_ref` appear on canceled/killed runs; `artifacts[].availability` is `available | partial | pruned` (pruned entries keep name + a truthful "content no longer stored" note).

Tones: `ok`, `needs_you`, `degraded`, `failed`. First matching verdict wins (approval > quarantine > request > failure > quota/backoff > running > terminal).

### `compozy loop events` — durable timeline + follow

```console
$ compozy loop events looprun-8f3ab2c1d4e5f607 --limit 3
SEQ  ROUND  EVENT
84   g1     step revisor-estilo succeeded (attempt 2, 2m31s)
85   g1     approval "aplicar-correcoes" opened
86   —      run status: running

$ compozy loop events looprun-8f3ab2c1d4e5f607 --after 86 --follow -o jsonl
{"seq":87,"kind":"gate_verdict","generation":1,"gate_id":"aplicar-correcoes","verdict":"approved","at":"2026-08-19T18:52:10Z"}
{"seq":88,"kind":"node_running","generation":1,"node_id":"sintetizador","session_id":"ses-e1b20f77","at":"2026-08-19T18:52:11Z"}
```

- Default view is `--view notable` (lifecycle, approvals, requests, gates, status). `--view all` is the ordered union of every tier — it adds activity and chatter (token ticks, channel messages). Repetitive chatter (heartbeat-class events) arrives coalesced as one `×N` entry whose `seq` is the last raw sequence it spans — resuming after it never replays the folded events.
- Positions are **per-run sequence numbers**, scoped by the run you name in the command: `--after <seq>` resumes exactly after that sequence in this run's history — no gaps, no duplicates. There is no cross-run position at the CLI. `--follow` ends cleanly when the run reaches a terminal outcome (exit 0).

### Settlement audit (front 3) — observable through existing verbs

After any terminal outcome, records are settled and the audit reason is queryable:

```console
$ compozy loop kill looprun-8f3ab2c1d4e5f607
Run looprun-8f3ab2c1d4e5f607 killed.

$ compozy task list --include-loop --loop-run looprun-8f3ab2c1d4e5f607
ID                                          STATUS    LOOP
loop.looprun-8f3ab2c1d4e5f607.coordinator   canceled  revisao-paralela · run
loop.looprun-8f3ab2c1d4e5f607.g1.node.sintetizador.0
                                            canceled  revisao-paralela · g1
# zero live records — settlement is part of the terminal transition

$ compozy task timeline loop.looprun-8f3ab2c1d4e5f607.coordinator --limit 1 -o json
{"events":[{"kind":"task.status_changed","to":"canceled",
            "reason":"loop_run_terminal","detail":"run killed by operator"}]}
```

Records repaired by the reconciliation sweep (crash windows, pre-existing orphans) carry `"reason":"reconciled_run_terminal"` instead — distinguishable from natural settlement.

## HTTP / UDS API

Identical route sets on HTTP and UDS. New/changed routes only; realistic values.

### `GET /api/tasks` — calm default + typed include

```
GET /api/tasks?workspace=demo
→ 200 {"items":[...only work items...],"facets":{...},"next_cursor":""}

GET /api/tasks?workspace=demo&include_loop=true&loop_run_id=looprun-8f3ab2c1d4e5f607
→ 200 {"items":[{"id":"loop.….coordinator","loop":{"run_id":"…","loop_name":"revisao-paralela","role":"coordinator"}}, ...]}
```

- `include_loop` (bool, default `false`) and `loop_run_id` (string, implies include) are typed query fields — never inferred from query text. Catalog items gain the optional `loop` object; facets and counts respect the same default.
- `parent_task_id=<coordinator-id>` returns cells regardless of `include_loop` (explicit parentage).
- The **single-task read** (`GET /api/tasks/:id`, CLI structured task read) carries the same optional `loop` object — deep links to a loop record never depend on list navigation. `loop_name` is optional: when the owning run was retention-deleted and the name is unrecoverable, it is omitted and the record reads "run no longer available".

### `GET /api/workspaces/:ws/loop-runs/:id/nodes` — run roster

```
GET /api/workspaces/demo/loop-runs/looprun-8f3ab2c1d4e5f607/nodes?state=all
→ 200 {"run_id":"…","nodes":[{...as CLI json above...}],"fanout_rollups":[...],"next_cursor":""}
```

Filters: `state` (`all|running|queued|waiting|retrying|paused|quarantined|succeeded|failed|canceled|not_taken`), `generation`, `cursor`, `limit`. Nodes the route never took return `"state":"not_taken"` with no fabricated timestamps.

### `GET /api/workspaces/:ws/loop-runs/:id/briefing`

```
→ 200 {"run_id":"…","status":"running","tone":"needs_you","headline":"…","detail":"…",
       "blockers":[{"kind":"approval","gate_id":"…","waiting_since":"…","unblocker":"…"}],
       "progress":{"round":1,"steps_done":4,"steps_total":6}}
```

### `GET /api/workspaces/:ws/loop-runs/:id/timeline` — durable paged story

```
GET …/timeline?view=notable&limit=50
→ 200 {"head_seq":90,
       "entries":[{"seq":90,"kind":"gate_verdict","generation":1,"title":"gate approved","at":"…"},
                  {"seq":84,"kind":"node_succeeded","generation":1,"node_id":"revisor-estilo",
                   "attempt":2,"title":"step revisor-estilo succeeded","at":"…"}],
       "next_cursor":"eyJydW4iOiJsb29w…IsImhlYWQiOjkwLCJiZWZvcmUiOjg0fQ"}
```

- **The first page is the newest window** and always returns `head_seq` — the run's highest sequence at that snapshot (`0` for a run with no events yet). Older history pages **backward on demand** via `next_cursor`; a long run never has to download its full history to become live.
- The HTTP page cursor is opaque and bound to `{run, view, snapshot head, before}` — the page set never shifts under concurrent appends, and replaying a cursor against another run (or a fork/rerun, which mint new run ids) returns `409 timeline_branch_changed` instead of splicing histories. (The CLI never handles this token — `--after` is a plain per-run sequence.)
- `view=notable` (default) | `all` (ordered union of notable + activity + chatter, with server-side coalescing of heartbeat-class runs of events). Pages are gap-free and duplicate-free.
- The existing SSE stream (`GET …/loop-runs/:id/events`) is unchanged and remains the low-latency push channel. Live readers take one first page, then attach the SSE stream with `after_sequence=head_seq` (works for `head_seq=0` on an empty run) — new events arrive by push while older history backfills on demand; the seam de-dupes by sequence.

## config.toml

```toml
[loops]
# How often the daemon sweeps for terminal runs with live execution records
# (settlement runs inline on every terminal transition; the sweep is the safety net).
reconcile_interval = "1m"
```

One new key on the existing `[loops]` section. `0s` is rejected at validation ("reconcile_interval must be positive"). Boot always runs one sweep regardless of interval.

## Native Tools

`compozy__task_list` gains the same typed fields:

```json
{"tool":"compozy__task_list","arguments":{"workspace":"demo","include_loop":true,"loop_run_id":"looprun-8f3ab2c1d4e5f607"}}
→ {"items":[{"id":"loop.….coordinator","loop":{"role":"coordinator","run_id":"…"}}],"next_cursor":""}
```

Default (`include_loop` absent) excludes loop records — agents see the same calm default as every other surface.

## Errors

| Condition | Surface | Exact result |
|---|---|---|
| Unknown run id | `loop why/nodes/events`, roster/briefing/timeline routes | `404 {"error":"loop_run_not_found","run_id":"…"}` · CLI exit 1, `Error: loop run "…" not found in workspace demo` |
| Invalid `--state` value | `loop nodes` | `400 {"error":"invalid_node_state","allowed":["all","running",…]}` · CLI exit 2 with the allowed list |
| `--state running` without `--run` | `loop nodes` | CLI exit 2, `Error: --state running requires --run (workspace inventory tracks exception states only)` |
| Timeline page cursor (opaque) from another run / after fork | timeline route | `409 {"error":"timeline_branch_changed"}` |
| `--after` beyond the run's history head | `loop events` | CLI exit 1, `Error: position 4200 is beyond this run's history (head: 90)` |
| Malformed cursor | list/roster/timeline | `400 {"error":"invalid_cursor"}` |
| `include_loop` non-boolean | tasks routes/tool | `400 {"error":"invalid_query_field","field":"include_loop"}` |
| Cross-workspace run id | all run-scoped routes | `404 loop_run_not_found` (scoping, never a leak) |
| Node verb on terminal run | existing node verbs (unchanged) | existing `409 loop_run_terminal` behavior — unchanged by this spec |
