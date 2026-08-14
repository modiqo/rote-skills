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
- Before returning reusable results, hand the adapter evidence and caller-supplied approval state to
  `rote-flow-crystallization`; it owns pending write/save and the save decision. Never run the
  emitted scaffold from this skill.

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
  conditions; play search has either been checked or explicitly delegated back to `rote` before new
  adapter exploration.
- Owns: delegated single-adapter workspace entry, probe/call/query sequence, cached response
  preservation, write-guard handling, and return summary for the main conversation. Does not own
  pending stub creation, save decisions, scaffold execution, release, index, search, or cleanup.
- Hands off to: `rote` when no installed adapter matches or the top-level route must change;
  `rote-workspace` when broader multi-adapter workspace orchestration is needed; `rote-registry` only
  for an already released artifact that needs sharing; `rote-flow-crystallization` for reusable
  evidence and the save gate.
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

If this is a delegated subagent task, do not run play search again. The caller already selected the
single-adapter route; resume that route and continue directly to sequential workspace commands.

If this is a fresh top-level task:

```bash
rote play search "<task intent>"
```

If a play matches, stop adapter exploration and run the play using the main `rote` skill's play
execution rules.

If no play matches, create and enter a workspace:

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
| 3 | Declarative fan-out, conditions, or reusable orchestration | Default no-shape-flag `rote play template create`; author `for_each`, conditions, and adapter calls in frontmatter `steps:` |
| 4 | API control flow the step language does not support — pagination that must run to exhaustion, data-dependent retry/termination, runtime method selection, or a stateful session/transaction protocol — or an explicit user request | Explicit `rote play template create ... --legacy-body` no-steps play |

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

For reusable results, return the evidence to `rote-flow-crystallization` before reusable result text.
Carry the user intent, workspace name/path, cached response IDs, adapter id, validated response paths,
result shape, caveats, and any caller-supplied save decision. Crystallization owns pending write,
pending save, and the accepted, declined, unclear, or pre-approved decision; never infer the decision
or execute the emitted scaffold here.

If the caller asks this delegated skill to return before crystallization, mark the save gate
`pending` and stop. On acceptance or pre-approval, `rote-flow-authoring` owns scaffold execution,
implementation, tests, lint, release, index/search verification, and pending cleanup.

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
- Commands run: probe/call/query commands
- Cached responses: `@N` ids and what each contains
- Write-guard state: none, approval required with token, or confirmed
- Result or artifact: ...
- Save gate: pending, accepted, discarded, or not applicable
- Next skill: `rote`, `rote-flow-crystallization`, `rote-registry`, or none
- Blockers: missing credential, unsafe write, adapter mismatch, or none
- Completion signal: task result delivered, approval requested, or next owner named
```

## TypeScript Play Boundary

In the default steps + presentation play, registered adapter effects belong in frontmatter `steps:`. The
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
`rote guidance typescript play-creation`.

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
