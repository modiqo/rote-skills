# TypeScript Transformations

Use this reference when cached rote responses need TypeScript transformation or a TypeScript flow body is being authored. `rote grammar deno` remains the source of truth for execution syntax.

## Execution rules

- Run legacy TypeScript flows with `rote deno run --allow-all`.
- Run `steps_with_presentation` flows with `rote flow run`; direct Deno skips the effect plane and
  does not provide presentation input.
- Run flow files from outside the active workspace.
- Do not call system `deno` directly.
- Do not prefix the binary with `~/.rote/bin/`; use `rote` on `PATH`.

Typical execution:

```bash
rote deno run --allow-all /absolute/path/to/main.ts [args]
```

## SDK imports

Use the SDK import form shown by `rote grammar deno` or `rote guidance typescript essential`. Avoid npm-style package assumptions unless the live guidance explicitly supports them for the current rote version.

For `steps_with_presentation`, do not import `sdk/ts/mod.ts` or construct `Rote`. Import
`__ROTE_PRESENTATION_SDK__`, then use `loadPresentationContext()`, literal `stepName("...")`
references, and `FlowOutput`.

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

Test with representative cached data or fixture input before release. Cover no-result, partial-result, and optional-parameter cases. After changes, rerun legacy TypeScript through `rote deno run --allow-all`; rerun `steps_with_presentation` through `rote flow run`.
