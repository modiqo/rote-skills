# Troubleshooting

Use this reference when a rote task hits a repeated failure mode. Keep commands sequential and prefer live guidance (`rote start`, `rote guidance`, `rote grammar`) before guessing.

## Missing model identity

Symptom: workspace commands complain about missing model/provider metadata or produce incomplete tracking.

Fix: enter the workspace and run `rote model set <model> --provider <provider>` before adapter calls. If the model cannot be inferred from the harness, ask one targeted question.

## Wrong flow path shape

Symptom: a flow path assembled by hand does not exist or misses an organization segment.

Fix: rerun `rote flow search "<intent>" --json` and use the returned `path` verbatim. Do not construct `~/.rote/flows/<name>/main.ts` manually.

## Wrong endpoint form

Symptom: `rote mcp init-session <id>` fails with an endpoint-not-found message, or rote suggests checking
available endpoints.

Fix: run `rote inventory`, then use the endpoint name exactly as shown. Adapter sessions normally use
`adapter/<id>` for `init-session`, while shorthand probe/call commands use the adapter id with hyphens
converted to underscores: `rote <id_with_underscores>_probe ...` and
`rote <id_with_underscores>_call ...`.

## Adapter auth mismatch

Symptom: catalog notes say an adapter should be public or unauthenticated, but calls fail with missing
token, bearer-token, or unauthorized errors.

Fix: compare `rote adapter catalog info <id>` with `rote adapter info <id>` and
`rote adapter keys <id>`. If rote says auth cannot be updated from inside a workspace, leave the
workspace and run the exact suggested `rote adapter update-auth <id> ...` command. Do not fall back to
curl or direct API calls just because generated auth defaults are wrong.

## Stale subagent workspace context

Symptom: a subagent cannot use cached `@N` responses or continues from the wrong workspace.

Fix: only delegate before workspace work starts. If delegation already happened, require the subagent to report workspace name, response IDs, result, and save-gate status before the main agent continues.

## Catalog-search drift

Symptom: no installed adapter matches, but direct MCP, WebFetch, curl, or scripts are tempting.

Fix: run `rote adapter catalog search "<intent>"` first. If a hit is useful, inspect or install it with rote before falling back out-of-band.

## jq pitfalls

Symptom: response queries fail due to shell expansion, quoting, or raw/string confusion.

Fix: quote jq filters with single quotes, use `-r` only when raw strings are wanted, and check `rote grammar query` for current response-address syntax.

Before changing jq syntax, inspect whether the cached response is an error envelope:

```bash
rote is-error @N && rote @N '$' -r
```

If the response is an adapter error, fix the upstream call, auth, base URL, or arguments before any
more `fromjson` queries. Retrying the same query against an error string only creates noisy failures.

## Blind retries

Symptom: the last command failed, or it already succeeded with warnings, and you are tempted to run
the same command again.

Fix: do not repeat it until something material has changed: command arguments, adapter configuration,
flow code, working directory, environment, or user input. If nothing changed, read the previous output
again and follow its `@@next`, `@@mandatory`, or error-recovery hint.

## Flow lint or release failures

Symptom: `rote flow lint` or `rote flow release` fails with runtime, static, or frontmatter errors.

Fix: change the flow or its configuration, then rerun lint/release. `rote flow validate` and
successful execution are useful checks, but they do not release the flow or make it searchable. Never
edit frontmatter `status` directly to bypass `rote flow release`.

## Shell expansion prompts

Symptom: commands behave differently because the shell expanded globs, variables, or quotes before rote saw them.

Fix: quote user-provided strings and jq filters. Run one rote command per tool call so any prompt, error, or `@@next` guidance is visible before continuing.

## Direct filesystem inspection pitfalls

Symptom: manual reads from workspace or flow directories conflict with rote mcp state.

Fix: inspect through rote commands. Direct filesystem reads can bypass cached responses, miss managed state, or assume a layout that changed between rote versions.
