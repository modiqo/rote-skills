---
name: rote-using-adapters
description: >
  Use when executing a task through one installed rote adapter, especially in a delegated
  subagent or adapter-specific helper. Applies after the main rote skill has selected an
  adapter for a single-adapter workflow.
---

# rote-using-adapters - Single-Adapter Execution

All `rote-<name>` references in this document — including every name in the Handoff
Contract — are companion **skills**, never CLI commands (`rote-shell` is not `rote shell`).
Invoke them through the runtime's skill mechanism; only literal `rote …` commands run in a
terminal.

Use this with the main `rote` skill. The main skill decides whether a task is single-adapter,
multi-adapter, or out-of-scope. This skill gives the execution protocol once one adapter is
selected.

Prefer `rote-workspace` for ordinary single-adapter execution in the main conversation. Use this
skill mainly for delegated adapter helpers or runtimes that explicitly select a single-adapter
execution specialist. It must still return through the shared workspace/crystallization lifecycle
before final presentation.

## Handoff Packet

Consume this packet from `rote`, `rote-task-routing`, or a generated adapter helper:

- Origin skill: `rote`, `rote-task-routing`, `rote-workspace`, or a generated helper.
- Target skill: `rote-using-adapters`.
- Adapter: `<adapter-id>`.
- Task: the user request being delegated.
- Existing workspace: path, only for resumptions.
- Cached responses: `@N` ids and their meanings, if any.
- Write-guard state: none, or confirmation token + impact + workspace.
- Save gate: not started, pending write done, save approved, or save declined.
- Stop conditions: missing credential, unsafe write, approval needed, or adapter mismatch.
- Return fields: workspace name/path, response IDs, user-visible result, write-guard state,
  reusable-result decision, next recommended skill.

## Core Rules

- Use only `rote` commands for adapter work. Do not call MCP servers, provider CLIs, raw
  adapter URLs, or external automation tools directly.
- Run commands sequentially. `@N` responses and `$variables` exist only after earlier commands
  finish.
- If starting fresh, create one workspace with `rote init <name> --seq` before probing or
  calling tools.
- If resuming, `cd` to the supplied workspace and do not run `rote init` again.
- Always probe before calling. Tool names and input schemas vary by adapter.
- Before returning reusable results, write a pending flow stub, run pending save, and route through
  `rote-flow-crystallization`. Run the emitted scaffold only after the caller or user approves
  saving, releasing, publishing, or making the workflow reusable.

## Auth Shape Check

If probe/call fails for missing or invalid auth, classify before advising:

```bash
rote adapter list <id> --json --health
```

- Do not assume Bearer means static token. OAuth, OAuth DCR / MCP PRM, and Google Discovery can
  all appear as bearer at the transport layer.
- Static bearer/API key: hand off to masked credential entry or `rote token set <ENV> --stdin`
  only as explicit terminal opt-in.
- OAuth client id/secret or OAuth DCR / MCP PRM: return a blocker recommending
  `rote adapter reauth <id>` or `rote adapter reauth <id> --scheme <name>`.
- Google Discovery: return a blocker recommending `rote oauth setup google --scopes ...`.
- Unknown bearer: report the health output and route to `rote-adapter-config`; do not request a
  pasted token.

## Handoff Contract

- Use when: `rote` or a router selected exactly one installed adapter to satisfy a delegated task,
  especially in a subagent or adapter-specific helper.
- Preconditions: the caller supplied adapter id, user intent, any existing workspace, and stop
  conditions; flow search has either been checked or explicitly delegated back to `rote` before new
  adapter exploration.
- Owns: delegated single-adapter workspace entry, probe/call/query sequence, cached response
  preservation, write-guard handling, pending flow stub creation, and return summary for the main
  conversation. Does not own release/index/search cleanup.
- Hands off to: `rote` when no installed adapter matches or the top-level route must change;
  `rote-workspace` when broader multi-adapter workspace orchestration is needed; `rote-registry`
  after a saved flow is ready to share; `rote-flow-crystallization` when the caller owns the save
  gate.
- Returns to: `rote` or the delegating skill with workspace path, cached response IDs, result,
  write-guard state, reusable-result state, and next recommended skill.
- Stop when: the task completes, probe shows the adapter cannot satisfy the request, a write guard
  needs confirmation, a credential is missing, or save/release/publish approval is unresolved.
- Completion signal: handoff summary produced or returned with workspace, response IDs, result,
  save gate, and blocker or next owner.

## Start

Run the protocol check first:

```bash
rote start
```

If this is a delegated subagent task, do not run flow search again. The caller already selected the
single-adapter route; resume that route and continue directly to sequential workspace commands.

If this is a fresh top-level task:

```bash
rote flow search "<task intent>"
```

If a flow matches, stop adapter exploration and run the flow using the main `rote` skill's flow
execution rules.

If no flow matches, create and enter a workspace:

```bash
rote init <adapter-id>-task --seq
cd ${ROTE_HOME:-$HOME/.rote}/rote/workspaces/<adapter-id>-task
rote model set <model> --provider <provider> --confirmed-current
```

If the environment cannot provide model/provider identity, continue after `rote start`; do not
invent values.

## Probe First

Find the exact operation and input schema:

```bash
rote <adapter_id>_probe "<natural language operation>"
```

Hyphens in the adapter id become underscores in the command name (`my-api` → `my_api_probe`).
Use the exact operation name from the probe result. Read required parameters, optional
parameters, and response shape before making the call.

Then call through rote:

```bash
rote <adapter_id>_call <operation_name> '{"param": "value"}' -s
```

Use `-s` when the adapter may need session state.

## Data Transformation Rules

Do not use Python, Node.js, Ruby, shell scripts, or raw `jq` subprocesses for filtering,
reshaping, or formatting data already captured in rote responses. Use rote-native extraction.
Do not pipe `rote query` output into `head`, `tail`, `grep`, `jq`, `python`, `node`, or temp files.
If an external command is truly required, run it through `rote proc run` so the workspace keeps
guidance and provenance.

| Tier | When | Use |
|------|------|-----|
| 1 | Field extraction, filtering, counts, sorting | `rote query @N '<jq>'` |
| 2 | Formatting that jq cannot express cleanly | `rote query @N --transform-ts '...'` or `--filter-ts` |
| 3 | Declarative fan-out, conditions, or reusable orchestration | Default no-shape-flag `rote flow template create`; author `for_each`, conditions, and adapter calls in frontmatter `steps:` |
| 4 | API control flow the step language does not support — pagination that must run to exhaustion, data-dependent retry/termination, runtime method selection, or a stateful session/transaction protocol — or an explicit user request | Explicit `rote flow template create ... --legacy-body` no-steps flow |

Start at Tier 1:

```bash
rote query @1 '.data[] | select(.status == "active")'
rote query @1 '.items | length'
rote query @1 '.users[] | {name, email}'
rote query @1 '.id' -s item_id
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

The last required lifecycle commands inside the workspace, before reusable result text, are pending
write followed by pending save. Pending save only prepares the scaffold command; it does not require
save approval.

```bash
rote flow pending write <workspace> \
  --name <suggested-flow-name> \
  --adapter <adapter-id> \
  --response-path "<validated jq path>" \
  --notes "<encoding quirks, caveats, or data shape notes>"

# Always prepare the scaffold command before returning reusable result text.
rote flow pending save <workspace>
```

Capture the scaffold command printed by `pending save`. If the user already asked to save, release,
publish, or make the workflow reusable, treat that as approval and run the scaffold path. Otherwise,
keep the workspace name and captured scaffold command, present the results, and ask:

```text
Want to save this as a reusable flow? (yes/no)
```

Do not create, release, or discard the flow until the user answers or the original request already
gave save, release, publish, or make-reusable approval. Never skip pending write or pending save.

## If User Saves The Flow

Once the user has approved saving, run the scaffold command yourself. Parameterize API inputs, not just pagination/output knobs.
For structured filters, add a raw JSON passthrough flag such as `--filter`.

Before release, verify:

```text
[ ] Adapter effects declared in frontmatter steps: (endpoint/method/params with $param tokens);
    only an explicit --legacy-body flow calls adapters from the TypeScript body
[ ] FlowOutput wired in the presentation body (legacy: in the imperative body):
    new FlowOutput(); out.human(...); out.summary(...); out.result({...})
[ ] Presentation body reads flow inputs from ctx.params, not Deno.args or step stdout
[ ] Frontmatter parameters match the named key=value contract (legacy: CLI flags)
[ ] Tests cover at least three distinct inputs, including one default-only run
[ ] rote flow lint <name> exits 0
```

Then use the release command; never edit `main.ts` to flip status manually:

```bash
rote flow release <name>
rote flow index --rebuild
rote flow pending discard <workspace>
```

## Handoff Summary

Return this summary to the caller when delegated adapter work completes, blocks, or needs save-gate
follow-up:

```markdown
# Rote Handoff Summary

- Active skill: `rote-using-adapters`
- Origin skill: `rote` or delegated helper
- Adapter: <adapter-id>
- User intent: ...
- Workspace path: ...
- Commands run: probe/call/query/pending-write commands
- Cached responses: `@N` ids and what each contains
- Write-guard state: none, approval required with token, or confirmed
- Result or artifact: ...
- Save gate: pending, accepted, discarded, or not applicable
- Next skill: `rote`, `rote-flow-crystallization`, `rote-registry`, or none
- Blockers: missing credential, unsafe write, adapter mismatch, or none
- Completion signal: task result delivered, approval requested, or next owner named
```

## TypeScript Flow Boundary

In the default steps + presentation flow, registered adapter effects belong in frontmatter `steps:`. The
`steps_with_presentation` body imports only `__ROTE_PRESENTATION_SDK__` and reads typed recorded
observations plus `ctx.params`; it must not call adapters, `fetch`, Deno file APIs, or
subprocesses. An adapter step is `endpoint` + `method` + `params` with `$param` substitution:

```yaml
steps:
  fetch_issues:
    endpoint: adapter/github
    method: issues/list-for-repo
    params: { owner: $owner, repo: $repo, per_page: $limit }
```

Step syntax quick reference: `rote grammar steps`; complete worked example:
`rote guidance typescript flow-creation`.

Only inside an explicitly scaffolded `--legacy-body` with no `steps:` use these imperative APIs:

| Operation | Do this | Not this |
|-----------|---------|----------|
| Read/write local files | `Deno.readTextFile` / `Deno.writeTextFile` | shell subprocess |
| Fetch public unauthenticated URL | `fetch(url)` | `curl` subprocess |
| Parse/transform data | TypeScript in memory | Python or jq subprocess |
| Call registered adapter API | `adapter.callBg("tool", params, { queue })` | raw fetch with auth |

## No Adapter Match

If no installed adapter can handle the task, return to the main `rote` skill's catalog search
and out-of-band fallback protocol. Do not silently bypass rote.
