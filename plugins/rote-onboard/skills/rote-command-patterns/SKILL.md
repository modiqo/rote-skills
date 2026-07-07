---
name: rote-command-patterns
description: >
  Use task-focused rote command idioms only after live `rote grammar` and `rote guidance` surfaces
  are checked. Covers adapter probes/calls, cached response queries, batches, queues, browser
  handoffs, registry pushes, and owner-skill returns.
---

# rote-command-patterns

All `rote-<name>` references in this document — including every name in the Handoff
Contract — are companion **skills**, never CLI commands (`rote-shell` is not `rote shell`).
Invoke them through the runtime's skill mechanism; only literal `rote …` commands run in a
terminal.

Use this skill when command syntax or rote idioms are needed for an active rote task and live help is
not enough by itself. Prefer live surfaces first: `rote start`, `rote guidance <area> essential`, and
`rote grammar <topic>`.

This skill does not own task execution. It returns command patterns to the skill that already owns
the workflow.

## Live Surfaces First

- Use `rote grammar query` for cached response addressing and jq flags.
- Use `rote grammar adapters` and `rote guidance adapters essential` before adapter probes, calls,
  sessions, or batches.
- Use `rote grammar deno` and `rote guidance typescript essential` before TypeScript flow execution.
- Use `rote grammar export` before flow scaffolding, pending flow work, lint, release, or index
  commands.
- Use `rote grammar registry` and `rote guidance registry essential` before pushes, pulls, namespace
  selection, or publish steps.

If live grammar conflicts with this skill, live grammar wins.

## Adapter Probe And Call

Start in `rote-workspace`, then inspect before calling:

```bash
rote <adapter_id>_probe "<operation intent>"
rote <adapter_id>_call <operation> '<args-json>'
```

Convert hyphenated adapter IDs to underscores in the command name. Read the full probe response,
copy the exact operation name and required JSON shape, then call through rote. Read `@@next`,
`@@mandatory`, `@@flows`, warnings, and cached response ids before continuing.

## Response Querying

Use cached response IDs instead of copying JSON:

```bash
rote query @1 '<jq-filter>' -r
```

Quote jq filters with single quotes unless the filter intentionally needs shell variables. Use `-r`
only when raw strings are wanted. If a query fails, inspect whether the cached response is an error
envelope before changing jq syntax.

Prefer rote-native, POSIX-friendly response methods before raw filesystem access:

- Use `rote query @N '<jq-filter>'` for extraction and `rote query is-error @N` for error checks.
- Use `rote grammar query` when syntax, raw output flags, or response addressing are unclear.
- Use rote-provided Deno/TypeScript transformations for complex reductions that do not fit jq.
- Keep transformations attached to response IDs so provenance remains visible in the workspace.

Do not read `.rote/responses/@N.json` directly for normal analysis, and do not pipe those internal
files through ad hoc scripts. Direct response-file inspection is a troubleshooting exception only:
use it after rote-native query or state inspection cannot recover the needed data, record why, and
return to rote commands as soon as the cause is known.

Prefer separate adapter/workspace lifecycle commands for setup, repair, pending save, release,
index, and registry verification so each output is read before the next step.
Captured shell command chains belong under `rote-shell`; use them only when each meaningful process
is run through `rote proc`.

## Batch Calls And Queues

Use batch patterns only after a single call proves the operation and input shape. Keep the batch in
rote so responses remain cached and queryable. For long-running or queued work, capture the task ID,
poll through rote commands, and query cached results when complete instead of managing background
processes directly.

## Browser And Registry Handoffs

For browser automation, hand off to `rote-browse` and use `rote guidance browser essential` for live
browser conventions. Keep browser-derived data flowing through rote workspace state before
crystallizing a reusable flow.

For registry publishing, use live registry guidance and grammar, confirm the target namespace and
artifact, then hand off to `rote-registry`. If organization administration is needed, hand off to
`rote-org` through `rote-registry`.

## Return Fields

Return these fields to the owning skill:

- Live grammar or guidance command checked.
- Pattern selected and exact command shape.
- Quoting, cwd, workspace, or response-id caveats.
- Owning skill to resume: `rote-workspace`, `rote-flow-authoring`, `rote-registry`, `rote-browse`, or
  `rote`.

## Handoff Contract

- Use when: an active rote workflow needs command idioms and live grammar/guidance alone is
  insufficient or needs task-specific interpretation.
- Preconditions: another skill owns the task branch, and the runtime can run `rote grammar` or report
  why live command help is unavailable.
- Owns: command-pattern lookup, live-surface precedence, quoting/cwd caveats, and return of exact
  command shapes to the owning skill.
- Hands off to: `rote-workspace` for adapter execution, `rote-flow-authoring` for flow lifecycle
  commands, `rote-typescript-transformations` for TypeScript execution detail, `rote-registry` for
  publish/pull commands, `rote-browse` for browser commands, and `rote-troubleshooting` for repeated
  failures.
- Returns to: the skill that requested command guidance with the selected pattern and live source of
  truth.
- Stop when: the owning skill has enough syntax to continue, live grammar supersedes this guidance,
  or repeated failures require troubleshooting.
- Completion signal: exact command pattern, caveats, and owning skill to resume are named.
