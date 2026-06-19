---
name: rote-using-adapters
description: >
  Use when executing a task through one installed rote adapter, especially in a delegated
  subagent or adapter-specific helper. Applies after the main rote skill has selected an
  adapter for a single-adapter workflow.
---

# rote-using-adapters - Single-Adapter Execution

Use this with the main `rote` skill. The main skill decides whether a task is single-adapter,
multi-adapter, or out-of-scope. This skill gives the execution protocol once one adapter is
selected.

**Inputs expected from the caller:**

```text
Adapter: <adapter-id>
Task: <user request>
Existing workspace: <path>   # only for resumptions
```

## Core Rules

- Use only `rote` commands for adapter work. Do not call MCP servers, provider CLIs, raw
  adapter URLs, or harness-native tools directly.
- Run commands sequentially. `@N` responses and `$variables` exist only after earlier commands
  finish.
- If starting fresh, create one workspace with `rote init <name> --seq` before probing or
  calling tools.
- If resuming, `cd` to the supplied workspace and do not run `rote init` again.
- Always probe before calling. Tool names and input schemas vary by adapter.
- Before returning results, write and save a pending flow stub.

## Start

Run the protocol check first:

```bash
rote start
```

If this is a fresh task:

```bash
rote flow search "<task intent>"
```

If a flow matches, stop adapter exploration and run the flow using the main `rote` skill's flow
execution rules.

If no flow matches, create and enter a workspace:

```bash
rote init <adapter-id>-task --seq
cd ${ROTE_HOME:-$HOME/.rote}/rote/workspaces/<adapter-id>-task
rote model set <model> --provider <provider>
```

If the environment cannot provide model/provider identity, continue after `rote start`; do not
invent values.

## Probe First

Find the exact operation and input schema:

```bash
rote <adapter_id_with_underscores>_probe "<natural language operation>"
```

Use the exact operation name from the probe result. Read required parameters, optional
parameters, and response shape before making the call.

Then call through rote:

```bash
rote <adapter_id_with_underscores>_call <operation_name> '{"param": "value"}' -s
```

Use `-s` when the adapter may need session state.

## Data Transformation Rules

Do not use Python, Node.js, Ruby, shell scripts, or raw `jq` subprocesses for filtering,
reshaping, or formatting data already captured in rote responses. Use rote-native extraction.

| Tier | When | Use |
|------|------|-----|
| 1 | Field extraction, filtering, counts, sorting | `rote @N '<jq>'` |
| 2 | Formatting that jq cannot express cleanly | `rote @N --transform-ts '...'` or `--filter-ts` |
| 3 | Loops, conditionals, reusable orchestration | `rote flow template create` TypeScript flow |

Start at Tier 1:

```bash
rote @1 '.data[] | select(.status == "active")'
rote @1 '.items | length'
rote @1 '.users[] | {name, email}'
rote @1 '.id' -s item_id
```

Only escalate when the lower tier cannot express the transformation.

## Write-Guard

If a call returns `confirmation_required`, stop and surface the token and workspace path to the
main conversation or user. Do not start over.

Return this shape:

```text
WRITE-GUARD APPROVAL REQUIRED
Tool: <tool_name>
Impact: <impact text>
Confirm token: <token>
Workspace: <workspace field from @@result>
```

After approval, re-enter the same workspace and retry the blocked call with
`--confirm <token>`. Do not run `rote start` or `rote init` on resume.

## Task Completion Protocol

The last two commands inside the workspace, before any result text, are pending write and
pending save.

```bash
rote flow pending write <workspace> \
  --name <suggested-flow-name> \
  --adapter <adapter-id> \
  --response-path "<validated jq path>" \
  --notes "<encoding quirks, caveats, or data shape notes>"

rote flow pending save <workspace>
```

Capture the scaffold command printed by `pending save`. Then present the results and ask:

```text
Want to save this as a reusable flow? (yes/no)
```

Do not create, release, or discard the flow until the user answers.

## If User Saves The Flow

Run the scaffold command yourself. Parameterize API inputs, not just pagination/output knobs.
For structured filters, add a raw JSON passthrough flag such as `--filter`.

Before release, verify:

```text
[ ] FlowOutput wired: new FlowOutput(); out.human(...); out.summary(...); out.result({...})
[ ] Frontmatter parameters match CLI flags
[ ] Tests cover at least three distinct inputs, including one default-only run
[ ] rote flow lint <name> exits 0
```

Then use the release command; never edit `main.ts` to flip status manually:

```bash
rote flow release <name>
rote flow index --rebuild
rote flow pending discard <workspace>
```

## TypeScript Flow Boundary

Inside TypeScript flows:

| Operation | Do this | Not this |
|-----------|---------|----------|
| Read/write local files | `Deno.readTextFile` / `Deno.writeTextFile` | shell subprocess |
| Fetch public unauthenticated URL | `fetch(url)` | `curl` subprocess |
| Parse/transform data | TypeScript in memory | Python or jq subprocess |
| Call registered adapter API | `adapter.callBg("tool", params, { queue })` | raw fetch with auth |

## No Adapter Match

If no installed adapter can handle the task, return to the main `rote` skill's catalog search
and out-of-band fallback protocol. Do not silently bypass rote.
