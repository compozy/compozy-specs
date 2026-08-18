# CompozyOS Specs

Specifications for [CompozyOS](https://github.com/compozy/compozy) — the agent operating system.

Each folder holds the document set behind one feature: what it does and why, the surfaces it exposes, the states it renders, the cases that prove it, and the decisions taken along the way. Specs are written before the code and stay the reference while the feature is built.

They live here rather than in the product repo so the runtime stays lean — and so the reasoning behind a change is readable without cloning the daemon.

## Index

Named `YYYY-MM-DD-<slug>`, dated by when the work began.

| Spec | What it covers | Documents |
| --- | --- | --- |
| [graph-eng](graph-eng/) · *in execution* | Loop Graph Completion: human interaction beyond approval, exclusive routing, fan-out completion strategies, operator time travel. [Tracking](https://github.com/compozy/compozy/issues/376) | `spec` |
| [2026-08-16-herdr-parity](2026-08-16-herdr-parity/) | Session attention: telling at a glance which of many concurrent sessions needs you, with a real notification path | `spec` |
| [2026-08-16-electron-shell](2026-08-16-electron-shell/) | Moving the desktop shell from Tauri's OS webview to Chromium, so every surface runs the same engine | `spec` |
| [2026-08-16-agent-plugins](2026-08-16-agent-plugins/) | Ingesting the portable [Agent Plugins 1.0.0](https://agent-plugins.org/) package format as extensions. [Tracking](https://github.com/compozy/compozy/issues/383) | `spec` |
| [2026-08-12-worktree-support](2026-08-12-worktree-support/) | Native git worktrees so parallel agents stop overwriting each other's uncommitted state | `prd + techspec` |
| [2026-08-10-desktop-app](2026-08-10-desktop-app/) | Shipping CompozyOS as a real desktop app instead of one more browser tab | `brief + prd + techspec` |
| [2026-08-06-remote-gateway](2026-08-06-remote-gateway/) | Unlocking the daemon from its machine: remote auth, inbound events, bridge delivery without self-hosted proxies | `prd + techspec` |
| [2026-08-02-loop-node-lifecycle](2026-08-02-loop-node-lifecycle/) | The inner contract of a Loop node: what one node does when the world fails around it | `prd + techspec` |
| [2026-08-02-bundles-removal](2026-08-02-bundles-removal/) | Hard-deleting the Bundle product so Extension is the only kit unit | `brief + techspec` |
| [2026-08-01-loops-paper-adoption](2026-08-01-loops-paper-adoption/) | Adopting the validated mechanisms from arXiv 2607.19297 into the Loop domain — and recording what was rejected | `techspec` |
| [2026-07-30-window-tabs](2026-07-30-window-tabs/) | Tabs for the desktop: many surfaces per window instead of a frame per surface | `prd + techspec` |
| [2026-07-29-ext-improvs](2026-07-29-ext-improvs/) | Making extensions authorable from outside this repo, and distributable without a gatekeeper | `brief + techspec` |
| [2026-07-29-cross-workspace-access](2026-07-29-cross-workspace-access/) | Opening a door in workspace isolation through the session permission mode operators already know | `techspec` |
| [2026-07-26-network-changes](2026-07-26-network-changes/) | Making Agent Network participation opt-in instead of implicitly enrolling ordinary work | `prd + techspec` |
| [2026-07-26-hermes-comparison](2026-07-26-hermes-comparison/) | Improvements drawn from a Hermes comparison: memory, tools, redaction, cost estimation | `techspec` |
| [2026-07-26-hermes-bridge](2026-07-26-hermes-bridge/) | Bridge usability parity: tool-call progress and one-shot delivery | `techspec` |
| [2026-07-26-compozy-migration](2026-07-26-compozy-migration/) | The hard cut from AGH to Compozy v0.3.0, closing the approved parity gaps | `brief + techspec` |
| [2026-07-25-modals-redesign](2026-07-25-modals-redesign/) | Landing the 16 designed entity editors, with the modal standard and state matrix behind them | `spec + techspec` |
| [2026-07-24-window-management](2026-07-24-window-management/) | Daemon-authoritative window management: virtual desktops, tiled groups, floating windows | `techspec` |
| [2026-07-24-os-shell](2026-07-24-os-shell/) | Turning the web experience from a one-page-at-a-time dashboard into an actual OS shell | `prd + techspec` |
| [2026-07-24-scheduler-capacity-starvation](2026-07-24-scheduler-capacity-starvation/) | Fixing a scheduler that treated a busy compatible worker as no worker at all | `techspec` |

## How a spec set is organized

The document set changed shape over time. Two generations appear here.

**Current (unified pipeline).** One spec with two parts, plus four companions:

| File | What it holds |
| --- | --- |
| `_spec.md` | **Part I** frames the product — overview, goals, features, business rules, user experience, non-goals. **Part II** designs the implementation — architecture, data models, endpoints, sequencing, safety invariants. |
| `_user_stories.md` | The canonical story catalog: every story with its acceptance criteria and edge cases. |
| `_dx.md` | The developer-experience contract: grammar, CLI verbs, HTTP/UDS routes, config keys, native tools, and the exact error each surface returns. |
| `_uiux.md` | The UI change map: every surface touched, what changes on it, which states must be designed. |
| `_tests.md` | The test contract: every case, its layer, and the story it proves. |

Read `_spec.md` first; the companions are linked from it where they matter.

**Earlier (split pipeline).** The same content, divided across separate documents rather than parts of one:

| File | What it holds |
| --- | --- |
| `_brief.md` | The framing that preceded the spec — the problem, the evidence gathered, the scope decision. Present when the work started from an open question. |
| `_prd.md` | The product side: overview, goals, user stories, features, business rules, non-goals. Equivalent to Part I above. |
| `_techspec.md` | The technical design: architecture, data models, endpoints, sequencing, invariants. Equivalent to Part II above. |

Read them in that order — `_brief.md` → `_prd.md` → `_techspec.md` — then the companions (`_user_stories.md`, `_tests.md`, and `_uiux.md` where the work touched the UI).

Not every spec carries every document. Some began as a technical decision with no separate product framing and carry only `_techspec.md`; the shape tells you how the work started. `2026-07-25-modals-redesign` sits across the change and carries both a `_spec.md` and a `_techspec.md`.

**Common to both generations.** `adrs/` holds the Architecture Decision Records — one per fork that was decided, with the alternatives that lost and why. `_tasks.md` plus `task_NN.md` is the execution plan: the task graph with its dependency edges, and one file per task carrying requirements, subtasks, assigned test cases, and the prior art its design draws on.

## Conventions

- **Specs are written, then executed.** A spec is approved before implementation starts; peer-review rounds are folded back into the document rather than tracked as separate revisions.
- **Non-goals are explicit.** Every spec names what it deliberately leaves out, so scope creep is visible rather than assumed.
- **Delete targets are named.** CompozyOS is pre-1.0 with no backward-compatibility debt; a spec that removes something lists the exact artifacts to delete.
- **Every capability is agent-operable.** A feature that only works through the web UI is incomplete, so each spec carries the CLI / HTTP / UDS / native-tool surface for its capabilities.

## Notes on reading the archive

- **AGH** is the product's earlier name. Specs written before `2026-07-26-compozy-migration` use it throughout; that spec is the rename itself.
- Supporting research (`analysis/`), QA evidence, review rounds, and run state stay in the product workspace and are not published here, so a few links inside older documents point at files you will not find.
- These documents are historical records of the decisions taken at the time. Where a spec and the shipped runtime disagree, the runtime is the truth.

## Contributing

Published for reading, not for external authorship. Corrections and questions belong on the tracking issue in [compozy/compozy](https://github.com/compozy/compozy/issues).
