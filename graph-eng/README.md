# Loop Graph Completion (graph-eng)

Closes the four gaps a cross-industry comparison found between the CompozyOS Loop engine and the graph-engineering state of the art: what humans can do mid-run, how paths are chosen, what fan-out joins can do, and what operators can do with history.

**Status:** in execution · **Tracking:** [compozy#376](https://github.com/compozy/compozy/issues/376)

## Read in this order

1. **[`_spec.md`](_spec.md)** — the spec. Part I is the product; Part II is the implementation design.
2. [`_user_stories.md`](_user_stories.md) — 26 stories (US-001..US-026) with acceptance criteria and edge cases.
3. [`_dx.md`](_dx.md) — the public surface: YAML grammar, CLI, HTTP/UDS, `config.toml`, native tools, error codes.
4. [`_uiux.md`](_uiux.md) — 11 web surfaces (S1–S11), their states, and their reference artboards.
5. [`_tests.md`](_tests.md) — 151 test cases mapped to the stories they prove.

## Execution plan

[`_tasks.md`](_tasks.md) is the task graph: 11 tasks with their dependency edges and the MVP boundary. The backend chain (01→07) is strictly serialized by append-only migration ordering — each task owns exactly one migration and none may ride another's. Web follows the backend; the QA pair closes.

| Task | Phase | Scope |
| --- | --- | --- |
| [01](task_01.md) | P0 | Cleanup: `hash_fields` deletion, predicate policy, per-gate counters |
| [02](task_02.md) | P1 | Router: `route` node + gate verdict routing |
| [03](task_03.md) | P2 | Requests core: `ask` node, respond plane, `ResponderPolicy` |
| [04](task_04.md) | P3 | Per-lane addressing, review gate, amend overlay |
| [05](task_05.md) | P4+P5 | Strategies, partial, progress namespace, iteration names |
| [06](task_06.md) | P6 | Windowed fan-out width |
| [07](task_07.md) | P7+P8 | Time travel: diff, rerun, fork + cross-cutting suites |
| [08](task_08.md) | P9 | Web run-page surfaces (S1–S8, S10–S11) + design pass |
| [09](task_09.md) | P10 | Bell composition (S9, post-herdr seam) |
| [10](task_10.md) | Tail | QA planning over the living `docs/qa` tree |
| [11](task_11.md) | Tail | QA execution: scenario walks + browser e2e |

Each task file carries its own requirements, subtasks, assigned test IDs, relevant files, and the prior-art citations its design draws on. The whole suite executes only after the herdr-parity program merges to `main` (ADR-006).

## Decisions

| ADR | Decision |
| --- | --- |
| [ADR-001](adrs/adr-001.md) | Human interaction model — four declarable interactions on the wait-resume plane |
| [ADR-002](adrs/adr-002.md) | Operator time travel — read-only diff, in-run rerun, cross-run fork |
| [ADR-003](adrs/adr-003.md) | Router node and gate-verdict routing, with a mandatory default route |
| [ADR-004](adrs/adr-004.md) | Fan-out v2 — completion strategies, first-class partial, progress, naming, windowed width |
| [ADR-005](adrs/adr-005.md) | Bug absorption — `hash_fields` deletion, predicate failure defaults, gate revision semantics |
| [ADR-006](adrs/adr-006.md) | Bell integration rides the herdr-parity attention pipeline |
