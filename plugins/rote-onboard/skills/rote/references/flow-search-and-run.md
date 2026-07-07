# Flow Search And Run

Use this reference when `rote flow search "<intent>"` returns a plausible reusable flow and you need
the shortest reliable path to run it.

1. Resolve the exact flow contract:

```bash
rote flow info <flow-name-or-path> --json
```

Use the returned `path` and declared parameters. Do not reconstruct a path from the name.

2. Pick the execution mode from the flow's frontmatter (or `rote flow info --json`).

Use `rote deno run --allow-all` for legacy `.ts` flows whose frontmatter has no `steps:` block,
passing positional args in declared order, from a directory outside the active workspace:

```bash
rote deno run --allow-all /absolute/path/to/main.ts [args in declared order]
```

Use `rote flow run` for any flow whose frontmatter has `steps:`, passing named `key=value`
parameters — direct Deno skips the effect plane:

```bash
rote flow run /absolute/path/to/main.ts [param=value ...]
```

`metadata.execution_model: steps_with_presentation` is still a `steps:` flow: the runner executes
the declared steps first, then invokes the presentation body. Do not run it through direct Deno —
the body would not receive the typed presentation input.

3. Verify the requested output, not just process success. Check the artifact path, required
sections, source markers, and live-data evidence the user asked for.

4. Stop after a verified full-flow match. Do not explore adapters, create a pending flow, or
rebuild the workflow unless the user requested new workflow work or the flow only covers a baseline.

5. For a partial match, preserve the baseline flow name, parameters, output artifact, and uncovered
requirements before routing the remaining work to the next rote skill.
