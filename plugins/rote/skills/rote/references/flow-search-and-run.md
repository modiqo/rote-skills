# Play Search And Run

Use this reference when `rote play search "<intent>"` returns a plausible reusable play and you need
the shortest reliable path to run it.

1. Resolve the exact play contract:

```bash
rote play info <play-name-or-path> --json
```

Use the returned `path` and declared parameters. Do not reconstruct a path from the name.

2. Pick the execution mode from the play's frontmatter (or `rote play info --json`).

Use `rote deno run --allow-all` for legacy `.ts` plays whose frontmatter has no `steps:` block,
after checking the captured `--help` text or a `--dry-run` invocation for the body's actual
argument syntax, from a directory outside the active workspace:

```bash
rote deno run --allow-all /absolute/path/to/main.ts [arguments verified from --help or --dry-run]
```

Use `rote play run` for any play whose frontmatter has `steps:`, passing named `key=value`
parameters — direct Deno skips the effect plane:

```bash
rote play run /absolute/path/to/main.ts [param=value ...]
```

`metadata.execution_model: steps_with_presentation` is still a `steps:` play: the runner executes
the declared steps first, then invokes the presentation body. Do not run it through direct Deno —
the body would not receive the typed presentation input.

3. Verify the requested output, not just process success. Check the artifact path, required
sections, source markers, and live-data evidence the user asked for.

4. Stop after a verified full-play match. Do not explore adapters, create a pending play, or
rebuild the workflow unless the user requested new workflow work or the play only covers a baseline.

5. For a partial match, preserve the baseline play name, parameters, output artifact, and uncovered
requirements before routing the remaining work to the next rote skill.
