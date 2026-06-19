# Flow Authoring

Use this reference when the user asks to create, edit, lint, release, verify, or publish a reusable rote flow. Keep live command help authoritative for syntax: `rote grammar export`, `rote grammar deno`, `rote guidance typescript essential`, and `rote guidance registry essential`.

## 1. Elicit the reusable contract

Before scaffolding, identify the repeatable workflow boundary:

- The user-visible goal and success condition.
- Required inputs, optional inputs, and safe defaults.
- Adapter operations needed to produce the result.
- Output shape a future caller should receive.
- Values that must remain parameters rather than hard-coded secrets, dates, IDs, or names.

If the requirement depends on an API shape, discover it through rote adapter probes and calls in a workspace before writing the flow.

## 2. Discover input schemas from APIs

Use rote commands to inspect adapter capabilities and response examples. Prefer cached response IDs and jq filters over copying large JSON into the prompt.

For each candidate parameter, record:

| Parameter | Required? | Source | Notes |
| --- | --- | --- | --- |
| User input | yes/no | Prompt, adapter schema, or default | Include validation or accepted shape. |

Keep raw passthrough as an escape hatch only when the adapter operation genuinely accepts an open-ended JSON body. Prefer named parameters for stable workflows.

For adapter-backed flows, enumerate the API input surface before scaffolding. Response shape shows
what came back; input schema shows what the reusable flow can ask for. Do not count runner knobs
such as `--max`, `--first`, pagination, or `--output` as meaningful reuse by themselves.

Hard rules for reusable adapter parameters:

- Probe first, then read the selected tool's input schema before choosing flags.
- For every server-side filter/input dimension, either expose a CLI flag, hardcode it with a short
  reason, or omit it with a short reason.
- If the API accepts a structured filter, where, query, body, or params object, include a raw JSON
  passthrough flag such as `--filter` or `--body` unless the live guidance says not to.
- Keep frontmatter parameters in lockstep with CLI flags: same names, types, defaults, required
  state, and descriptions that mention the underlying API field.
- Test different server-side dimensions, not only the same pagination/output knob with different
  values.

## 3. Scaffold through rote

Use the pending save command from [flow-crystallization.md](./flow-crystallization.md) when this authoring work came from a completed task. This still applies when the original user request already said to save or release the flow: run pending write/save first, then run the emitted scaffold command. For direct authoring, use the current scaffold/export syntax from `rote grammar export`.

`rote flow template create` is flag-driven: pass `--name <flow-name>`, repeated `--adapter <adapter-id-or-endpoint>` flags, and `--workspace <workspace-name>` when scaffolding from workspace history. Do not pass the flow name as a positional argument unless live rote grammar says that syntax has returned.

Do not create flow directories by hand unless live rote guidance says to. The CLI owns the current layout, frontmatter, and index metadata.

## 4. Implement the flow body

For TypeScript flows, follow [typescript-transformations.md](./typescript-transformations.md) and `rote grammar deno`.

Preserve a stable `FlowOutput` shape:

- Return structured data for machine reuse.
- Include a concise human-readable summary when useful.
- Keep adapter raw responses out of the final result unless the user needs them.
- Avoid embedding local workspace paths, secrets, or one-off IDs in output.

## 5. Test with diverse inputs

Run the flow from `/tmp` with representative parameter sets:

```bash
cd /tmp && rote deno run --allow-all /absolute/path/to/main.ts [args]
```

Cover the common case, an empty or no-result case, optional-parameter defaults, and at least one user-provided edge case. For shell flows, run the shell entrypoint directly from `/tmp`.

## 6. Lint, release, and rebuild search

Before calling the work complete, run the live lint/release path surfaced by rote. Use `rote grammar export` and `rote guidance typescript essential` for current commands.

Release is a hard gate, not a file edit. Execution success and `rote flow validate` do not make a
flow discoverable; only `rote flow release <name>` performs the local lifecycle transition.

Before release, obtain explicit authorization unless the original request already asked to release,
crystallize, finalize, make discoverable, save as reusable, or publish the flow. For create, edit,
lint, or verify requests without that approval, report the tested draft state and ask whether to run
the release/index/search sequence.

Once release is approved, use this sequence:

```bash
cd /tmp && rote deno run --allow-all /absolute/path/to/main.ts [representative args]
rote flow lint <name>
rote flow release <name>
rote flow index --rebuild
rote flow search "<intent-or-flow-name>"
rote flow search "<intent-or-flow-name>" --json
```

If lint or release fails, fix the violation before retrying. Do not rerun the same failing command
until flow code, arguments, adapter configuration, cwd, or environment has changed. Do not edit
frontmatter `status` by hand; manual status flips skip lint, release events, and search metadata.

The search result must expose the released path and parameter contract a future agent can run through [flow-search-and-run.md](./flow-search-and-run.md).

## 7. Registry handoff

If the flow should be shared, use the registry guidance and namespace push patterns from `rote guidance registry essential`, `rote grammar registry`, and [command-patterns.md](./command-patterns.md). Confirm the target namespace before pushing.
