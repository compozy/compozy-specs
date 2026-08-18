# CompozyOS Specs

Public specifications for [CompozyOS](https://github.com/compozy/compozy) — the agent operating system.

Each spec is a folder holding the full document set behind one feature: what it does and why, the surfaces it exposes, the states it renders, and the cases that prove it. Specs are written before the code and stay the reference while the feature is built.

They live here rather than in the product repo so that the runtime stays lean — and so anyone can read the reasoning behind a change without cloning the daemon.

## Index

| Spec | Status | Tracking |
| --- | --- | --- |
| [graph-eng](graph-eng/) — Loop Graph Completion: human interaction, routing, fan-out strategies, operator time travel | In execution | [compozy#376](https://github.com/compozy/compozy/issues/376) |

## How a spec set is organized

| File | What it holds |
| --- | --- |
| `_spec.md` | The spec itself. **Part I** frames the product — overview, goals, features, business rules, user experience, non-goals. **Part II** designs the implementation — architecture, data models, endpoints, sequencing, safety invariants. |
| `_user_stories.md` | The canonical story catalog: every user story with its acceptance criteria and edge cases. Part I links to it; tests trace back to it. |
| `_dx.md` | The developer-experience contract: grammar, CLI verbs, HTTP/UDS routes, config keys, native tools, and the exact error codes each surface returns. |
| `_uiux.md` | The UI change map: every surface the feature touches, what changes on it, which states must be designed, and its reference artboard. |
| `_tests.md` | The test contract: every case, its layer, and the story it proves. |
| `adrs/` | Architecture Decision Records — one per fork that was decided, with the alternatives that lost and why. |

Start at `_spec.md`. The other four are its companions, and the spec links into each where it matters.

## Conventions

- **Specs are written, then executed.** A spec is approved before implementation starts; peer-review rounds are folded back into the document rather than tracked as separate revisions.
- **Non-goals are explicit.** Every spec names what it deliberately leaves out, so scope creep is visible rather than assumed.
- **Delete targets are named.** CompozyOS is pre-1.0 with no backward-compatibility debt; a spec that removes something lists the exact artifacts to delete.
- **Every capability is agent-operable.** A feature that only works through the web UI is incomplete, so each spec carries the CLI / HTTP / UDS / native-tool surface for its capabilities.

## Contributing

These documents are published for reading, not for external authorship. Corrections and questions belong on the tracking issue in [compozy/compozy](https://github.com/compozy/compozy/issues) linked from the index above.
