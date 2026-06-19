# rote reference map

Read only the reference needed for the current branch of the task. `SKILL.md` keeps the gates
that must always stay in context; this directory holds task-specific detail.

## Reading Table

| If you are about to... | Start here |
| --- | --- |
| Run a matched existing flow | [flow-search-and-run.md](./flow-search-and-run.md) |
| Choose no-flow task routing or subagent handoff | [task-routing.md](./task-routing.md) |
| Execute adapter calls in a workspace | [workspace-protocol.md](./workspace-protocol.md) |
| Recover a pending stub or ask save/discard | [flow-crystallization.md](./flow-crystallization.md) |
| Create, lint, release, verify, or publish a flow | [flow-authoring.md](./flow-authoring.md) |
| Look up task-focused command idioms | [command-patterns.md](./command-patterns.md) |
| Transform cached responses or write TypeScript flow logic | [typescript-transformations.md](./typescript-transformations.md) |
| Diagnose repeated rote workflow failures | [troubleshooting.md](./troubleshooting.md) |

## Reading Rules

- Keep `../SKILL.md` in context for the direct-adapter stop rule, sequential command rule,
  flow-search-first routing, catalog fallback, and pending save gate.
- Load a reference only after the task reaches that branch.
- Prefer live rote command surfaces over copied syntax when a topic is covered by
  `rote guidance <area> essential` or `rote grammar <topic>`.
- Do not load every reference preemptively; progressive disclosure keeps the active task context small.
