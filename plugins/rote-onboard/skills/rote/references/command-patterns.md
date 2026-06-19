# Command Patterns

Use this reference for concise rote command idioms after the task branch is known. Prefer live surfaces for exact syntax: `rote start`, `rote guidance <area> essential`, and `rote grammar <topic>`.

## Adapter probe and call

Start in a workspace from [workspace-protocol.md](./workspace-protocol.md), then inspect before calling:

```bash
rote <adapter_id_with_underscores>_probe "<operation intent>"
rote <adapter_id_with_underscores>_call <operation> '<args-json>'
```

Convert hyphenated adapter IDs to underscores in the command name. Read the full probe response,
copy the exact operation name and required JSON shape, then call through rote. Read any `@@next` or
`@@flows` hints before the next command.

## Response querying

Use cached response IDs instead of copying JSON:

```bash
rote @1 '<jq-filter>' -r
```

Use `rote grammar query` when filters, raw output, or response addressing are uncertain. Avoid shell expansion surprises by quoting jq filters with single quotes unless the filter intentionally needs shell variables.

## Batch calls

Use batch patterns only after a single call proves the operation and input shape. Keep the batch in rote so responses remain cached and queryable. Check `rote guidance adapters essential` and `rote grammar adapters` for the current batch form.

## Background queues

For long-running or queued work, prefer rote's background/task guidance over direct process management. Capture the task ID, poll through rote commands, and query the cached result when complete.

## Browser handoff

For browser automation, load the companion `rote-browse` skill when available and prefer `rote guidance browser essential` for live browser conventions. Keep any browser-derived data flowing back through rote workspace state before crystallizing a reusable flow.

## Registry namespace push

Use registry live guidance for namespace auth, dry-run, and push syntax:

```bash
rote guidance registry essential
rote grammar registry
```

Confirm the namespace and artifact being pushed before publishing. If organization administration is needed, hand off to the `rote-org` companion skill.
