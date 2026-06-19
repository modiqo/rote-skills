# Workspace Protocol

Use this reference for no-flow tasks that need adapter calls, response queries, or transformations.
The workspace preserves rote mcp state, cached responses, and model identity for the run.

## 1. Initialize and enter the workspace

Before changing directories, record the user's artifact directory: the current working directory
where requested files such as `./report.md` should land. The rote workspace directory is for cached
responses and adapter state; it is not automatically the user's deliverable directory.

If the user gave a relative artifact path before workspace setup, resolve it against the original
artifact directory or pass it explicitly to the flow/adapter code. Do not accidentally write user
deliverables under `${ROTE_HOME}/rote/workspaces/<workspace-name>` just because you `cd` there for
rote commands.

Create a task-specific workspace with sequential execution enabled:

```bash
rote init <workspace-name> --seq
```

Enter the workspace before adapter calls:

```bash
cd ${ROTE_HOME:-$HOME/.rote}/rote/workspaces/<workspace-name>
```

Keep rote commands one-at-a-time. Read `@@status`, `@@next`, `@@flows`, cached `@N` responses, and
errors before the next command.

## 2. Set model identity before calls

Set the model once in the workspace before task execution:

```bash
rote model set <model> --provider <provider> --confirmed-current
```

If the active runtime, UI, prompt, user, or harness exposes a subject model, use that exact model
string and provider rather than guessing a nearby version. Do not infer it from subagent
frontmatter, harness defaults, or examples. If no current model identity is visible, skip this step
rather than fabricating metadata.

## 3. Probe, call, and query through rote

Use live help for current syntax:

- `rote guidance agent essential` for workspace workflow conventions.
- `rote guidance adapters essential` for adapter probe and call patterns.
- `rote grammar query` for cached response and jq usage.

Typical sequence:

```bash
rote <adapter_id_with_underscores>_probe "<operation intent>"
rote <adapter_id_with_underscores>_call <tool-or-operation> '<args-json>'
rote @1 '<jq-filter>' -r
```

Use the adapter shorthand commands that rote installs for each adapter. Convert hyphenated adapter
IDs to underscores in the command name, for example `example-api` becomes `example_api_probe` and
`example_api_call`. Probe first, read the exact tool name and input schema, then call that tool with
JSON arguments. Add `-s` or `--session` when the operation needs session state.

Use cached `@N` response IDs instead of copying large JSON between commands. If a response suggests
a flow through `@@flows`, pause adapter exploration and return to [flow-search-and-run.md](./flow-search-and-run.md).

## 4. Inspect state through rote commands

Inspect workspace state with rote commands, not direct filesystem reads. Prefer live command
surfaces such as `rote start`, `rote guidance`, and `rote grammar` when syntax is uncertain.

Direct filesystem inspection can miss rote-managed state, break across workspace layouts, or bypass
the response cache the next command needs.

Use this rote-native recovery checklist before reading files under `${ROTE_HOME}` directly:

```bash
rote workspace ls
rote ls
rote workspace inspect meta
rote workspace inspect variables
rote adapter list
rote flow pending list
rote flow list
rote sdk status
rote deno status
```

If the SDK is missing, run `rote sdk install` instead of searching `${ROTE_HOME}/lib` by hand. If a
flow inventory is needed, use `rote flow list`; `rote flow search ""` is not an inventory command and
may return no matches even when local flows exist.

After compaction, interruption, or a context handoff, recover state through rote before continuing:

```bash
rote workspace ls
cd ${ROTE_HOME:-$HOME/.rote}/rote/workspaces/<workspace-name>
rote ls
rote workspace inspect meta
rote workspace inspect variables
rote flow pending list
rote adapter list
rote flow list
```

Preserve the workspace name, cached `@N` response IDs, pending flow name, output artifact path,
and any release/index/search verification already completed. Do not start a new workspace unless the
rote mcp state commands show that no prior workspace exists.

If a distraction, compaction, or handoff happened after `rote flow pending write`, run
`rote flow pending list` and then `rote flow pending save <workspace-name>` again to re-emit the
scaffold command before continuing flow authoring. Do not guess the workspace or skip directly to
`rote flow template create` from memory.

## 5. Subagent re-entry and write-guard continuity

If a subagent was chosen before workspace work, it must initialize or enter its own rote workspace,
set model identity, and use rote commands sequentially. It should return:

- The workspace name.
- Important cached response IDs and what they contain.
- The user-visible result.
- Whether the result is reusable and needs the pending-stub save gate.

The main agent owns the final user interaction. Before presenting reusable results, continue with
[flow-crystallization.md](./flow-crystallization.md) and preserve the explicit yes/no save decision.

## 6. Finish workspace execution

When the task result is ready, do not immediately present it if the work is reusable. First write a
pending stub and prepare the save decision using [flow-crystallization.md](./flow-crystallization.md).
