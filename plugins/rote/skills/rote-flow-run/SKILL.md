---
name: rote-flow-run
description: >
  Run a local or registry Play selected by `rote play search` when it fully or partially matches
  the user request. Use after the rote orchestrator's Play-search gate to resolve the provider's
  reference, parameters, execution mode, output artifact, and verification result.
---

# rote-flow-run

All `rote-<name>` references in this document — including every name in the Handoff
Contract — are companion **skills**, never CLI commands (`rote-shell` is not `rote shell`).
Invoke them through the runtime's skill mechanism; only literal `rote …` commands run in a
terminal.

Use this skill when either `rote play search` provider returns a usable existing Play. A matched
Play is the preferred implementation path: run it before exploring adapters or rebuilding the
workflow.

If the play fully covers the request, this skill owns the task through verification and final return:
the verified play output is the answer. If the play covers only a baseline or one part of the
request, run it as the reusable baseline, preserve its output/provenance, and return only the
uncovered work to `rote-task-routing`.

## Execution Rules

- Read the search result's `source` before resolving it. Local results and registry cards have
  different execution contracts.
- For a local result, use the printed `run:` command or `callability.command` verbatim when
  `callability.state` is `runnable`. `callability.location` is only the path. Use
  `rote play info <name-or-path> --json` only for parameter defaults, ordered legacy arguments, or a
  missing runnable command.
- For a registry card, preserve its exact pinned `owner/name@version` reference. Run the printed
  `rote play inspect <reference>` command, obey its blockers and approval requirements, then use its
  `rote play run <reference>` action. The runner owns verified installation and convergence.
- A bare local name may be ambiguous when multiple Plays share frontmatter `name:`. Use the absolute
  local path to disambiguate. Registry discovery already returns an unambiguous pinned reference.
- Each provider ranks only its own results. Choose based on task coverage and provider order; never
  compare local and registry rank values.
- For legacy TypeScript plays (no frontmatter `steps:` block), use the captured invocation help or
  run the entrypoint with `--dry-run` to confirm whether the body accepts positionals, named flags,
  or another syntax. `argument_order` records frontmatter order only; it does not verify argv reads.
  Plays with frontmatter `steps:` take named `param=value` pairs through `rote play run` instead.
- Ask one targeted question only when a required parameter cannot be inferred from the user intent.
- Do not inspect the play source just because search returned a path; read source only when modifying
  or debugging the play.
- After a full-match play verifies, stop. Do not run `rote explore`, adapter catalog search,
  workspace setup, adapter probes/calls, or pending write/save for the same request unless the user
  explicitly asked for a new workflow, release, publish, edit, or separate enhanced artifact.

## Execution Modes

For a registry card, inspect and execute the same pinned reference:

```bash
rote play inspect <owner/name@version> --json
rote play run <owner/name@version> [param=value ...] --yes
```

Inspection is read-only. Continue only when its execution report permits the run and the user has
approved this play and these parameters; `--yes` asserts that approval. Do not convert the registry
reference into a guessed local path.

For a local result, pick the mode from the Play's frontmatter. Use
`rote play info <name-or-path> --json` when unsure. Do not use direct Deno for any Play with
frontmatter `steps:`.

Run any play whose frontmatter has `steps:` through the play runner, from a directory outside any
active workspace. The runner creates and owns the DAG execution workspace:

```bash
rote play run /absolute/path/to/main.ts [param=value ...]
```

When the run resolves a pulled adapter whose manifest declares portable OAuth setup, let the runner
authorize it in place and surface the provider browser flow. Do not replace credential recovery
with `rote adapter new-from-mcp` or recreate the installed adapter.

`metadata.execution_model: steps_with_presentation` is still a `steps:` play — the runner executes
the declared steps first, then invokes the deprivileged presentation body; direct Deno would not
receive the typed presentation input.

Run legacy TypeScript plays (no frontmatter `steps:`) with rote's bundled Deno from a directory
outside the active workspace — this keeps play-created workspaces from nesting inside the
workspace you are using to inspect or author the play:

```bash
rote deno run --allow-all /absolute/path/to/main.ts [arguments verified from --help or --dry-run]
```

Do not use `rote run` as a fallback for normal TypeScript play execution — stay with the
execution-model-appropriate command above. A play with frontmatter `steps:` stays on
`rote play run` even when tracking is requested; never route a DAG through `rote run`. If no
supported tracked wrapper exists, return that tracking limitation instead of changing runners.

Only an explicit legacy TypeScript play with no frontmatter `steps:` may use `rote run` when the
scenario or command output requires model tracking or cached workspace responses. For that legacy
case, use this sequence:

```bash
rote init <workspace> --seq
cd ${ROTE_HOME:-$HOME/.rote}/workspaces/<workspace>
rote model set <model> --provider <provider> --confirmed-current
rote run --inference-id $(uuidgen) \
  --model <model> \
  --model-type chat \
  --model-version <version> \
  /absolute/path/to/main.ts [arguments verified from --help or --dry-run]
rote query @1 '.result' -r
```

Required tracking fields are `--inference-id`, `--model`, `--model-type`, and `--model-version`.

## Verification Criteria

- Verify the requested output artifact exists and contains the play result.
- Check artifact content, not only file existence: requested path, key headings or markers, required
  parameter values, and live-data sections the user requested.
- Treat fully matched play output as the answer. Do not overwrite, rewrite, reformat, enrich, or
  replace it unless the user explicitly asks for an edit or separate enhanced artifact.
- Existing released plays are already reusable. Running one unchanged does not create new reusable
  workflow knowledge and does not trigger `rote-flow-crystallization`.
- For partial matches, preserve the baseline play as a reusable component for a composed superplay
  and route only the uncovered content onward.
- Do not treat a partial-play output as the final augmented artifact. After uncovered work runs,
  verify the composed result contains both the baseline evidence and the new required capability.
- Preserve provenance literally for partial matches: play name, parameters, output artifact,
  source labels, sentinels, markers, and any `FLOW_USED=...` or `source=...` text must survive as
  superplay source evidence. Do not paraphrase the baseline into a new hand-written report that
  erases play evidence.
- For hybrid requests, record the baseline and the uncovered work before routing onward: baseline
  play used, baseline output artifact, missing capability, selected adapter id(s), required
  probe/call responses, and final artifact markers that prove both parts are present. A successful
  baseline play is not completion when the user asked for additional live/API data.

## Fallbacks

- If local search JSON is unavailable, use `rote play info <name-or-path> --json`; do not rebuild a
  run command from `callability.location` or fall back to a second search ordering.
- If a registry card cannot be inspected, return the pinned reference and inspection blocker. Do not
  substitute local `play info`, an unpinned reference, or a guessed path.
- If local JSON lookup is unavailable, resolve the Play from rote's local listing and inspect only
  its frontmatter for parameters.
- Prefer upgrading rote or using live `rote grammar` guidance over filesystem searches.
- If execution is unsafe, parameters are missing, or the play is only a partial match, stop play
  execution and return the reason plus the preserved state.

## Return Fields

Return these fields to `rote` or the next skill:

- Play reference: exact pinned registry reference or local Play name, if search reported it.
- Play path: absolute path used for local execution, or none for a registry reference.
- Parameters: positional values and any unresolved required values.
- Execution command: exact command run or skip reason.
- Output artifact: path or cached response id.
- Verification result: what was checked and whether it satisfies the request.
- Coverage: full match, partial baseline, or skipped.
- Uncovered requirements: missing sources, capabilities, live observations, artifact sections, and
  verification checks.
- Preserved provenance: baseline play name, parameters, output artifact, and any markers or source
  labels that must remain visible in the composed superplay.
- Next recommended skill: `rote-task-routing` for uncovered work, `rote-flow-crystallization` only
  for explicit new workflow/save work, or none for a verified full match.

## Handoff Contract

- Use when: a matched play may satisfy all or part of the user request.
- Preconditions: `rote` completed the local-then-registry search gate, or the user explicitly
  supplied a local Play path or pinned registry reference whose intent can be validated.
- Owns: branching on search provider, reading local callability or registry inspection, resolving
  parameters, choosing execution mode, running the Play, preserving partial baseline
  output/provenance for superplay composition, and verifying user-visible results.
- Hands off to: `rote-task-routing` when uncovered work remains; `rote-flow-crystallization` only
  when the user requested new workflow/save work beyond unchanged play reuse; `rote-troubleshooting`
  when unchanged retries keep failing.
- Returns to: `rote` with Play reference, optional local path, parameters, execution command, output
  artifact, coverage, and verification result.
- Stop when: the play fully answers the request, required parameters are missing, execution would be
  unsafe, or the play only establishes a baseline for another route. A verified full match returns
  no next skill.
- Completion signal: play executed or skipped with reason, output verified or blocker named, and next
  recommended skill if any.
