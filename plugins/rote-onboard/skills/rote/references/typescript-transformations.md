# TypeScript Transformations

Use this reference when cached rote responses need TypeScript transformation or a TypeScript flow body is being authored. `rote grammar deno` remains the source of truth for execution syntax.

## Execution rules

- Run TypeScript flows with `rote deno run --allow-all`.
- Run from `/tmp` when executing flow files.
- Do not call system `deno` directly.
- Do not prefix the binary with `~/.rote/bin/`; use `rote` on `PATH`.

Typical execution:

```bash
cd /tmp && rote deno run --allow-all /absolute/path/to/main.ts [args]
```

## SDK imports

Use the SDK import form shown by `rote grammar deno` or `rote guidance typescript essential`. Avoid npm-style package assumptions unless the live guidance explicitly supports them for the current rote version.

## Transform cached responses

When a transformation can be expressed with jq, prefer `rote @N '<jq-filter>' -r`. Use TypeScript when the task needs richer validation, grouping, date handling, joins, or formatting than jq should carry.

Keep transformations deterministic:

- Treat missing fields explicitly.
- Keep user-provided parameters separate from constants.
- Return a structured result plus a concise summary when useful.
- Avoid reading rote workspace files directly; use response IDs and SDK helpers.

## FlowOutput shape

Design `FlowOutput` for future agents as well as humans:

- `summary` for the short answer.
- `data` or domain-specific fields for machine-readable results.
- `warnings` for partial or best-effort outcomes.
- No secrets, local paths, or transient workspace IDs unless the user asked for them.

## Testing transformations

Test with representative cached data or fixture input before release. Cover no-result, partial-result, and optional-parameter cases. After changes, rerun the flow through `rote deno run --allow-all` rather than a standalone TypeScript runner.
