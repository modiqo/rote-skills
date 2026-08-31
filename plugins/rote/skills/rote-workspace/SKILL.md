---
name: rote-workspace
description: >
  Execute adapter work inside a rote workspace after routing selects workspace execution. Use for
  workspace init/entry, sequential command discipline, model identity, cached response preservation,
  subagent re-entry, handoff packets, and markdown handoff summaries.
---

# rote-workspace

All `rote-<name>` references in this document — including every name in the Handoff
Contract — are companion **skills**, never CLI commands (`rote-shell` is not `rote shell`).
Invoke them through the runtime's skill mechanism; only literal `rote …` commands run in a
terminal.

Use this skill for no-play or partial-play tasks that need adapter calls, response queries, or
transformations. The workspace preserves rote state, cached responses, and model identity for the
run.

Do not use this skill to improve, rerun, or replace a verified full-play match. Workspace execution
starts only when no existing play covers the request, or when a partial play leaves explicit
uncovered work.

## Handoff Packet

Consume this packet from `rote`, `rote-task-routing`, or a delegated skill:

- Origin skill: `rote` or `rote-task-routing`.
- User intent: the exact result the user requested.
- Play baseline: play output/artifact to preserve, or none.
- Selected route: single adapter, multi-adapter orchestration, or catalog-installed adapter.
- Requirements: required sources/adapters, capabilities, live observations, output artifact,
  and checks that must survive handoff.
- Workspace name: proposed task-specific workspace name, if already chosen.
- Artifact directory: original user directory for requested output files.
- Cached responses: response ids already created and what each contains.
- Allowed commands: exact rote probes/calls/queries or adapter ids to use.
- Stop conditions: missing credential, unsafe action, failed precondition, or approval needed.
- Return fields: workspace path, commands run, response ids, result, reusability signal, save gate,
  and next recommended skill.

## Know the Requirements

Before adapter calls, know what makes the task complete: which capability each required source
must provide (not just the provider name), which live observation or artifact section is required,
and which cached `@N` response or command will prove each one.

Re-check those requirements after compaction, interruption, or a subagent return. If a cached
response lacks a required capability, search or route for the missing capability instead of
filling the gap with nearby metadata. If adapter config or troubleshooting changes anything, rerun
the failed probe/call before composing or verifying the artifact.

## 1. Initialize and Enter the Workspace

Before changing directories, record the user's artifact directory: the current working directory
where requested files such as `./report.md` should land. The rote workspace directory is for cached
responses and adapter state; it is not automatically the user's deliverable directory.

If the user gave a relative artifact path before workspace setup, resolve it against the original
artifact directory or pass it explicitly to the play/adapter code.

Create a task-specific workspace with sequential execution enabled:

```bash
rote init <workspace-name> --seq
```

Workspace commands resolve the active workspace from cwd, and command runners rarely preserve cwd
between steps — prefix every workspace-scoped command with the resolved workspace path as one
logical step:

```bash
cd ${ROTE_HOME:-$HOME/.rote}/workspaces/<workspace-name> && rote <command…>
```

Keep rote commands one at a time. Read `@@status`, `@@next`, `@@plays`, cached `@N` responses, and
errors before the next command.

## 2. Set Model Identity When Known

Set the model once in the workspace immediately after `rote init` and before `rote init-session`,
adapter probes, adapter calls, or play crystallization:

```bash
rote model set <model> --provider <provider> --confirmed-current
```

If the active runtime, UI, prompt, user, or harness exposes a subject model, use that exact model
string and provider. If no current model identity is visible, skip this step rather than fabricating
metadata. Do not skip this step when the model and provider are visible; it is part of workspace
identity, not an optional annotation.

## 3. Probe, Call, and Query Through Rote

Use live help for current syntax:

- `rote guidance agent essential` for workspace workflow conventions.
- `rote guidance adapters essential` for adapter probe and call patterns.
- `rote grammar query` for cached response and jq usage.

Typical sequence:

```bash
rote <adapter_id>_probe "<operation intent>"
rote <adapter_id>_call <tool-or-operation> '<args-json>'
rote query @1 '<jq-filter>' -r
```

Use adapter shorthand commands that rote installs for each adapter. Convert hyphenated adapter IDs
to underscores in the command name. Probe first, read the exact tool name and input schema, then call
that tool with JSON arguments. Add `-s` or `--session` when the operation needs session state.

Use cached `@N` response IDs instead of copying large JSON between commands. If a response suggests
a play through `@@plays`, pause adapter exploration and return to `rote-flow-run`.

For response inspection and transformation, stay inside rote first:

- Use `rote query @N '<jq-filter>' -r` and the exact `primary_query` or
  `response_path` surfaces printed by rote.
- If `rote query` returns nothing, do not inspect `.rote/responses` directly.
  Run `rote ls`, `rote query schema @N`, and retry against `@` when the latest response is likely intended.
  Use `rote guidance query essential` for cached response parsing patterns.
- Use `rote proc run` through `rote-shell` for local programs that meaningfully transform,
  validate, sort, join, or format data for the final artifact or future play.
- Do not pipe `rote query` output into `head`, `tail`, `grep`, `jq`, `python`, `node`, or temp files.
  If a non-rote command is unavoidable, route it through `rote proc run` instead.
- Do not use raw `grep`, `head`, `sort`, inline Python, temporary files, or shell pipelines to
  parse rote responses when the result feeds a user artifact, reusable play, or later reasoning
  step.
- Tiny disposable terminal inspection is acceptable only when it does not influence the final
  answer, report, play body, or verification claim.

## 4. Inspect and Recover State Through Rote

Inspect workspace state with rote commands, not direct filesystem reads. Prefer live command
surfaces such as `rote start`, `rote guidance`, and `rote grammar` when syntax is uncertain.

`rote workspace ls` is inventory, not an enterability check. Enter only rows it renders as
`active` or `empty`; those are the enterable `Complete` entries. Stop on every `Needs attention`
row — including incomplete, corrupt, mid-restore, unavailable, or responses-unreadable entries —
and follow its diagnosis or recovery hint instead of running `cd` or `rote ls` against it.

Use this rote-native recovery checklist before reading files under `${ROTE_HOME}` directly:

```bash
rote workspace ls
cd ${ROTE_HOME:-$HOME/.rote}/workspaces/<complete-workspace-name>
rote ls
rote workspace inspect meta
rote workspace inspect variables
rote adapter list
rote play pending list
rote play list
rote sdk status
rote deno status
```

After compaction, interruption, or a context handoff, recover state through rote before continuing:

```bash
rote workspace ls
cd ${ROTE_HOME:-$HOME/.rote}/workspaces/<complete-workspace-name>
rote ls
rote workspace inspect meta
rote workspace inspect variables
rote play pending list
rote adapter list
rote play list
```

Preserve the workspace name, cached `@N` response IDs, pending play name, output artifact path, any
partial-play baseline, and any release/index/search verification already completed. When recovering
the same task after compaction, interruption, handoff, or subagent return, reuse the named workspace
only when the inventory shows an enterable `Complete` row. For an unrelated task, start
`rote init <task-specific-name> --seq`; do not select a workspace merely because `rote workspace ls`
lists one. Export and crystallization consume recorded workspace history, so cross-task reuse
contaminates the resulting play.

Before final presentation, run `cd <workspace-path> && rote ls` — it surfaces the workspace's
`@@` state and emits the `[MANDATORY PROTOCOL]` pending-stub warning when a reusable result has
no pending play yet. Then verify the requirements in prose or with rote queries:

- Every required capability has evidence from the selected adapter, play, browser capture, or
  `rote proc` result.
- The output artifact includes every required source, not only the easiest successful response.
- Any baseline play output remains labeled and preserved when the task is a superplay.
- Any adapter repair has been followed by a rerun of the failed probe/call.
- Reusable work has either entered the pending save lifecycle or is explicitly not reusable.
  Run reuse triage before marking a save gate `not applicable`.

## 5. Subagent Re-Entry

If a subagent was chosen before workspace work, it must initialize or enter its own rote workspace,
set model identity when known, and use rote commands sequentially. It should return:

- Workspace name and path.
- Important cached response IDs and what they contain.
- User-visible result and output artifact.
- Whether the result is reusable and needs the pending-stub save gate.

The main agent owns the final user interaction. Before presenting reusable results, hand off to
`rote-flow-crystallization` and preserve the explicit yes/no save decision.

## Handoff Summary

Write or return this summary when workspace work must survive handoff, compaction, or subagent
return:

```markdown
# Rote Handoff Summary

- Active skill: `rote-workspace`
- Origin skill: `rote` or `rote-task-routing`
- User intent: ...
- Workspace path: ...
- Artifact directory: ...
- Play baseline: ...
- Requirements: ...
- Commands run: ...
- Cached responses: ...
- Current gate: workspace execution, recovery, save gate, or blocker
- Result or artifact: ...
- Save gate: pending, accepted, discarded, or not applicable
- Next skill: `rote-flow-crystallization`, `rote-troubleshooting`, `rote-registry`, or none
- Blockers: ...
- Completion signal: ...
```

## Finish Workspace Execution

When the task result is ready, do not immediately present reusable work. First prepare the
save/discard decision through `rote-flow-crystallization`. If the original request already asked to
save, release, publish, crystallize, make discoverable, or make the workflow reusable, mark save as
pre-approved in the handoff. Crystallization still completes applicable pending write/save; on
acceptance or pre-approval, `rote-flow-authoring` owns authoring, release, index, search
verification, and pending cleanup without asking again.

One-off describes the current need; reusable describes the procedure. Do not use one-off user intent
as the reason to skip the save gate.

This save gate applies to new workspace output, not to unchanged reuse of an existing released play.
If the only available result is a verified full-play artifact, return it to `rote` with save gate
`not applicable`.

If the result is not reusable, return the workspace path, response IDs, and user-visible artifact to
`rote` for final presentation. If the result might be reusable and you are unsure, hand off to
`rote-flow-crystallization`; do not silently skip the save gate.

## Handoff Contract

- Use when: routed work needs adapter calls, cached response queries, transformations, workspace
  state, or subagent re-entry.
- Preconditions: `rote-task-routing` or an equivalent owner has selected workspace execution and
  provided the user intent, route decision, artifact directory, and any play baseline to preserve.
- Owns: workspace init/entry, model identity when known, sequential rote command execution, cached
  response preservation, requirement preservation, rote-native state recovery, subagent
  workspace re-entry, and workspace handoff summaries.
- Hands off to: `rote-flow-run` when `@@plays` suggests a usable play; `rote-flow-crystallization`
  when output is reusable or might be reusable; `rote-troubleshooting` after repeated unchanged
  failures; `rote-registry` only for an already released artifact that needs sharing.
- Returns to: `rote` or the delegating skill with workspace path, commands run, cached response IDs,
  result/artifact, reusability signal, save gate, blockers, and next recommended skill.
- Stop when: the workspace result is complete, required approval or credentials are missing, rote
  state cannot be recovered, or a more specific skill becomes the correct owner before new calls run.
- Completion signal: workspace summary produced or returned, cached response IDs preserved, and final
  result, blocker, or next skill named.
