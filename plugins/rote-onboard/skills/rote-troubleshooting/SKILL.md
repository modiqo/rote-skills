---
name: rote-troubleshooting
description: >
  Diagnose repeated rote workflow failures without blind retries. Use after unchanged retries,
  missing model identity, wrong paths/endpoints, auth mismatches, jq errors, stale workspace context,
  lint/release failures, or direct-filesystem pitfalls.
---

# rote-troubleshooting

All `rote-<name>` references in this document — including every name in the Handoff
Contract — are companion **skills**, never CLI commands (`rote-shell` is not `rote shell`).
Invoke them through the runtime's skill mechanism; only literal `rote …` commands run in a
terminal.

Use this skill when a rote task hits a repeated failure mode, an unchanged retry just failed, or the
next step is unclear after reading `@@status`, `@@next`, `@@mandatory`, warnings, and errors. Keep
commands sequential and prefer live guidance (`rote start`, `rote guidance`, `rote grammar`) before
guessing.

This skill changes the cause, route, inputs, configuration, or owner. It never repeats the same
failing command unchanged.

Use this skill after two failures in the same class even if the exact command changed. Same-class
failures include wrong cwd/workspace, wrong endpoint, missing auth, bad jq/query, wrong adapter
capability, flow lint/runtime errors, and process evidence not being captured.

## Recovery Loop

1. Name the owning skill that failed and the exact command or gate that failed.
2. Read the previous output again, including structured markers and cached response IDs.
3. Classify the failure with the patterns below or live `rote guidance`.
4. Change one material input: arguments, adapter configuration, auth, cwd, workspace, flow code,
   environment, approval, or route owner.
5. Return to the owning skill once the cause changes or the route changes.

Stop and ask one targeted question only when a missing credential, destructive approval, or model
identity cannot be inferred.

Troubleshooting is not a terminal state. After changing config, route, cwd, SDK state, or flow code,
return to the owner with the failed command or gate to rerun. Do not final-answer from this skill
unless the result is a blocker that prevents any safe rote path.

## Common Failure Patterns

| Symptom | Recovery |
| --- | --- |
| Missing model identity | In the workspace, run `rote model set <model> --provider <provider> --confirmed-current` when the current model/provider is known. If unknown, skip rather than fabricate unless the command requires it. |
| Wrong flow path shape | Rerun `rote flow search "<intent>" --json` and use the returned `path` verbatim. Do not construct flow paths by hand. |
| Wrong endpoint form | Run `rote inventory`, then use endpoint names exactly as shown. Adapter sessions usually use `adapter/<id>`; shorthand commands use the adapter id with hyphens converted to underscores. |
| Adapter auth mismatch | Compare `rote adapter catalog info <id>`, `rote adapter info <id>`, and `rote adapter keys <id>`. Leave the workspace for adapter auth updates when rote says to. |
| Adapter base URL mismatch | Inspect installed adapter config before catalog install. If the installed provider-equivalent adapter has the wrong base URL, route to `rote-adapter-config`, repair it, then rerun the original probe/call. |
| Stale subagent workspace context | Do not delegate after workspace calls start. Require workspace name, response IDs, result, and save-gate status before continuing. |
| Catalog-search drift | Run `rote adapter catalog search "<intent>"` before direct MCP, WebFetch, curl, or scripts when no installed adapter fits. |
| Capability mismatch | Return to `rote-task-routing`, name the missing operation, and search or inspect adapters by capability. Do not keep using an adapter that only provides adjacent metadata. |
| jq pitfalls | Quote jq filters with single quotes, use `-r` only for raw strings, and check `rote grammar query`. Inspect whether the cached response is an error envelope before changing filters. |
| Flow lint or release failures | Change flow code, frontmatter, arguments, cwd, or environment before retrying. Never edit frontmatter `status` directly. |
| `rote deno run` runtime or config failures | Inspect `rote deno status`, `rote sdk status`, `rote grammar deno`, cwd, `ROTE_HOME`, import map, and SDK install state. Repair with `rote deno install`, `rote sdk install`, or `rote sdk upgrade` as appropriate. Do not switch to raw `deno`, `$ROTE_HOME/bin/deno`, or `/tmp/.../bin/deno`; that bypasses rote runtime setup, SDK imports, and provenance. If generated code imports from `$HOME/.rote` while `ROTE_HOME` is set elsewhere, fix the import to honor `ROTE_HOME` and rerun through `rote deno run`. |
| Shell expansion prompts | Quote user-provided strings and jq filters. Run one rote command at a time so prompts and errors are visible. |
| Untracked process evidence | Switch to `rote-shell`: rerun through `rote proc run`, capture files/stdout/stderr, and query the saved response before using the result in an artifact or completion claim. |
| Direct filesystem inspection pitfalls | Inspect workspace state and cached responses through rote commands before reading managed files directly. Direct `.rote/responses/@N.json` reads are allowed only when rote-native query/state commands cannot expose the data needed to diagnose the failure; record that reason and return to rote commands once recovered. |

## State Inspection Commands

Pick the group that matches the failure class — never walk the whole list hoping something turns
up:

- Guidance/syntax failures: `rote start`, `rote guidance agent essential`, `rote grammar <topic>`.
- Workspace/cached-response failures: `rote workspace ls`, then from inside the workspace
  `rote ls`, `rote workspace inspect meta`, `rote workspace inspect variables`.
- Adapter failures: `rote adapter list`.
- Flow lifecycle failures: `rote flow pending list`, `rote flow list`.
- Runtime/SDK failures: `rote sdk status`, `rote deno status`.

For cached response errors, inspect the response before changing transformation code:

```bash
rote query is-error @N && rote query @N '$' -r
```

If those commands fail or the workspace metadata itself is corrupted, direct file inspection can be
used as a last-resort diagnostic, not as the normal data path.

## Return Fields

Return these fields to the owning workflow skill:

- Owning skill and failed command or gate.
- Failure class and evidence from output or live guidance.
- Material change made or recommended.
- Route change, if any.
- Remaining blocker or targeted user question.
- Skill to resume: `rote-flow-run`, `rote-task-routing`, `rote-workspace`, `rote-flow-crystallization`,
  `rote-flow-authoring`, `rote-command-patterns`, `rote-typescript-transformations`, `rote-registry`,
  `rote-browse`, or `rote`.

## Handoff Contract

- Use when: a rote command, workflow gate, flow lifecycle, workspace operation, browser route, or
  registry operation fails repeatedly or would otherwise be retried unchanged.
- Preconditions: the failing owner, command/gate, previous output, and attempted inputs are known or
  can be recovered from conversation, workspace state, or rote commands.
- Owns: failure classification, no-blind-retry discipline, recovery command selection, route changes,
  and return-to-owner handoff once the cause changes.
- Hands off to: the owning workflow skill after recovery; `rote-command-patterns` when syntax is the
  blocker; `rote-adapter-config` when adapter settings must change; `rote-setup` when installation,
  login, or credentials are the blocker.
- Returns to: the skill that encountered the failure with cause, material change, remaining blocker,
  and resume point.
- Stop when: the cause changes and the owner can retry, a safer route is selected, a targeted user
  approval/credential is needed, or no rote path remains.
- Completion signal: changed input, changed route, named blocker, or owning skill resume point.
