# Loop Graph Completion (graph-eng)

Closes the four gaps a cross-industry comparison found between the CompozyOS Loop engine and the graph-engineering state of the art: what humans can do mid-run, how paths are chosen, what fan-out joins can do, and what operators can do with history.

**Status:** in execution · **Tracking:** [compozy#376](https://github.com/compozy/compozy/issues/376)

## Read in this order

1. **[`_spec.md`](_spec.md)** — the spec. Part I is the product; Part II is the implementation design.
2. [`_user_stories.md`](_user_stories.md) — 26 stories (US-001..US-026) with acceptance criteria and edge cases.
3. [`_dx.md`](_dx.md) — the public surface: YAML grammar, CLI, HTTP/UDS, `config.toml`, native tools, error codes.
4. [`_uiux.md`](_uiux.md) — 11 web surfaces (S1–S11), their states, and their reference artboards.
5. [`_tests.md`](_tests.md) — 151 test cases mapped to the stories they prove.

## Decisions

| ADR | Decision |
| --- | --- |
| [ADR-001](adrs/adr-001.md) | Human interaction model — four declarable interactions on the wait-resume plane |
| [ADR-002](adrs/adr-002.md) | Operator time travel — read-only diff, in-run rerun, cross-run fork |
| [ADR-003](adrs/adr-003.md) | Router node and gate-verdict routing, with a mandatory default route |
| [ADR-004](adrs/adr-004.md) | Fan-out v2 — completion strategies, first-class partial, progress, naming, windowed width |
| [ADR-005](adrs/adr-005.md) | Bug absorption — `hash_fields` deletion, predicate failure defaults, gate revision semantics |
| [ADR-006](adrs/adr-006.md) | Bell integration rides the herdr-parity attention pipeline |
