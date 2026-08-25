# Play Search And Run

Use this reference when either Play-search provider returns a plausible reusable Play and you need
the shortest reliable path to run it.

Search is local by default. Start there for installed Plays. When no verified full local Play fits,
or when the task is specifically registry discovery, search published Plays explicitly:

```bash
rote play search "<intent>" --source registry --json
```

Registry search defaults to `--scope accessible`: anonymous callers see public Plays, while a
usable signed-in session also includes private personal Plays and private Plays from every
organization the caller can access. Use `--scope public` when the result will feed a public page or
another identity-independent artifact. Use `--owner <slug>` only to narrow either scope. Registry
results are pinned discovery cards, not local callability records. Preserve the exact
`owner/name@version` reference and inspect it before execution:

```bash
rote play inspect <owner/name@version> --json
```

When inspection reports an executable Play and its blockers are resolved, present that inspection,
get the user's approval, then let the registry-aware runner verify, install, converge, and execute it
(`--yes` asserts the approval you just obtained):

```bash
rote play run <owner/name@version> [param=value ...] --yes
```

Do not send a registry card through `rote play info`, invent a local path, or compare its rank with
a local result's rank.

1. For a local result, use its typed callability first:

```bash
rote play search "<intent>" --json
```

Run `data.search_results[0].items[].callability.command` verbatim when it is present and
`state: runnable`. Use `rote play info <play-name-or-path> --json` only when the result is
blocked, lacks a command, or a legacy argument contract needs confirmation; do not reconstruct a
path from the name.

2. If a lookup is needed, pick the execution mode from the play's frontmatter.

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
