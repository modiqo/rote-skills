# rote-onboard - installed skill map

This file is the plugin and install-facing map for the bundled rote skill set. It helps humans,
installers, and tests sanity-check what shipped together. Active task routing starts in
`rote/SKILL.md`; do not depend on this file as automatically loaded runtime context.

Companion skills carry their own required gates and `## Handoff Contract` sections. The detailed
active graph and packet shape live in `rote/references/skill-workflow-map.md`, which `rote/SKILL.md`
or a companion skill loads explicitly only when needed.

## Active entrypoint

Use `rote` for day-to-day orchestration. It owns the rote skill rule, the flow-search-first gate,
platform reference selection, and lifecycle handoffs to narrower skills. When the task is already
clearly setup, update, registry, org, browser, adapter creation, or adapter configuration, the
matching companion skill can activate directly. Shell/process work routes through `rote-shell`.
Ordinary single-adapter execution still routes
through `rote-workspace` unless a generated helper or runtime explicitly selects
`rote-using-adapters`.

## Bundled skills

| Skill | Package role | Active owner for | Typical handoff |
| --- | --- | --- | --- |
| `rote` | Entrypoint orchestrator. | Flow search, top-level routing, platform adaptation, standard handoff packet. | Any companion skill; daily execution starts with `rote-flow-run`, `rote-task-routing`, or `rote-workspace`. |
| `rote-flow-run` | Daily execution path. | Matched-flow execution, parameter mapping, partial-baseline preservation, output verification. | `rote-task-routing`, `rote-flow-crystallization`, `rote`. |
| `rote-task-routing` | Daily execution path. | Explore/catalog gates, subagent-before-workspace decisions, adapter route selection, fallback boundary. | `rote-flow-run`, `rote-workspace`, `rote-using-adapters`, `rote-adapter-create`, `rote`. |
| `rote-workspace` | Daily execution path. | Workspace init/entry, sequential adapter commands, cached responses, model identity, handoff summaries. | `rote-flow-crystallization`, `rote-troubleshooting`, `rote-registry`, `rote`. |
| `rote-shell` | Shell/process path. | Local CLI commands, files, logs, process state, dependency checks, and shell-derived flow replay. | `rote-flow-authoring`, `rote-flow-crystallization`, `rote-troubleshooting`, `rote`. |
| `rote-flow-crystallization` | Reuse save path. | Pending write/save gates, save-or-discard decisions, and reusable-result recovery. | `rote-flow-authoring`, `rote-registry`, `rote`. |
| `rote-flow-authoring` | Flow authoring path. | Reusable contract elicitation, schema discovery, scaffold, tests, lint, release, search verification, and pending cleanup. | `rote-typescript-transformations`, `rote-registry`, `rote-flow-run`, `rote`. |
| `rote-command-patterns` | Command idiom path. | Task-focused rote command patterns after live grammar/guidance surfaces are checked. | Owning workflow skill, `rote-troubleshooting`. |
| `rote-typescript-transformations` | TypeScript transformation path. | Cached response transformations, TypeScript flow bodies, `FlowOutput` shape, and tests. | `rote-flow-authoring`, `rote-workspace`. |
| `rote-troubleshooting` | Recovery path. | Repeated-failure diagnosis, no-blind-retry recovery, route changes, and owner-skill resume points. | Original owning workflow skill, `rote`. |
| `rote-setup` | Guided setup path. | Install, login, first adapters, credentials, value proof. | `rote-adapter-create`, `rote-adapter-config`, `rote-registry`, `rote`. |
| `rote-adapter-create` | Adapter creation path. | Catalog/spec discovery, dry-run, auth research, create and probe. | `rote-adapter-config`, `rote-registry`, `rote-setup`, `rote`. |
| `rote-adapter-config` | Adapter tuning path. | Show/confirm/apply/re-show configuration loop. | `rote-adapter-create`, `rote`. |
| `rote-registry` | Sharing and registry path. | Adapter/flow push, visibility, org selection, invites, pull readiness. | `rote-org`, `rote-adapter-create`, `rote`. |
| `rote-org` | Organization administration path. | Orgs, members, roles, invites, usage, plan details. | `rote-registry`. |
| `rote-browse` | Browser automation path. | Browser sessions, snapshots, page slices, headed/headless flows. | `rote`, browser flow crystallization paths. |
| `rote-using-adapters` | Delegated single-adapter execution path. | Generated helper or runtime-selected work after `rote` has selected one installed adapter. | `rote-workspace`, `rote-flow-crystallization`, or `rote` for lifecycle completion. |
| `rote-update` | Update path. | Binary update, bundled skill refresh, restart guidance. | `rote-setup`, `rote`. |

## Installed reference layout

`rote/references/` contains support files only:

- Platform/tool mappings: `claude-code-tools.md` (Claude Code), `codex-tools.md` (Codex),
  `copilot-tools.md` (Copilot CLI), `gemini-tools.md` (Gemini), `pi-tools.md` (Pi), and
  `antigravity-tools.md` (Antigravity).
- Workflow support: `skill-workflow-map.md`, loaded only when the detailed companion graph or
  standard handoff packet is needed, and `flow-search-and-run.md`, the shortest search → info →
  run → verify path. Day-to-day task logic lives in standalone skill directories, not hidden
  files under `rote/references/`.
