---
name: rote-flow-run
description: >
  Run a flow returned by `rote flow search` when it fully or partially matches the user request.
  Use after the rote orchestrator's flow-search-first gate to resolve the flow path, parameters,
  execution mode, output artifact, and verification result.
---

# rote-flow-run

All `rote-<name>` references in this document — including every name in the Handoff
Contract — are companion **skills**, never CLI commands (`rote-shell` is not `rote shell`).
Invoke them through the runtime's skill mechanism; only literal `rote …` commands run in a
terminal.

Use this skill when `rote flow search "<intent>"` returns a usable existing flow. A matched flow is
the preferred implementation path: run it before exploring adapters or rebuilding the workflow.

If the flow fully covers the request, this skill owns the task through verification and final return:
the verified flow output is the answer. If the flow covers only a baseline or one part of the
request, run it as the reusable baseline, preserve its output/provenance, and return only the
uncovered work to `rote-task-routing`.

## Execution Rules

- If search returned a usable flow name, resolve that single flow with
  `rote flow info <name> --json`. If search returned or the user supplied a flow file path, use
  `rote flow info <path> --json`. This is the canonical path and parameter contract.
- Prefer the `execution.command` and `execution.argument_order` fields from `rote flow info --json`
  over any command template copied from search output or memory.
- A bare name may be ambiguous when multiple flows share frontmatter `name:`. Use an org-qualified
  id, such as `<org>/<name>`, or the absolute path from search/info to disambiguate. A path to an
  existing non-flow file should fail as "not a flow", not as a missing name.
- Use the `path` from `rote flow info` verbatim; do not construct
  `~/.rote/flows/<org>/<name>/main.ts` by hand.
- Treat `rote flow search --json` as discovery, not as the final single-flow contract. Search returns
  ranked fuzzy matches; info resolves one exact flow record and avoids client-side filtering.
- Map parameters positionally in the declared order for legacy TypeScript flows (no frontmatter
  `steps:` block) — their bodies read positional `Deno.args`, so named pairs arrive as literal
  `key=value` strings. Do not pass `key=value` pairs to them unless the flow documentation
  explicitly expects them, and do not convert parameter names into flags such as `--city` or
  `--output_path` for TypeScript flows that declare ordered positional parameters. Flows with
  frontmatter `steps:` take named `param=value` pairs through `rote flow run` instead.
- Ask one targeted question only when a required parameter cannot be inferred from the user intent.
- Do not inspect the flow source just because search returned a path; read source only when modifying
  or debugging the flow.
- After a full-match flow verifies, stop. Do not run `rote explore`, adapter catalog search,
  workspace setup, adapter probes/calls, or pending write/save for the same request unless the user
  explicitly asked for a new workflow, release, publish, edit, or separate enhanced artifact.

## Execution Modes

Pick the mode from the flow's frontmatter — inspect it or run
`rote flow info <name-or-path> --json` when unsure. Do not use direct Deno for any flow with
frontmatter `steps:`.

Run any flow whose frontmatter has `steps:` through the flow runner, from a directory outside any
active workspace. The runner creates and owns the DAG execution workspace:

```bash
rote flow run /absolute/path/to/main.ts [param=value ...]
```

`metadata.execution_model: steps_with_presentation` is still a `steps:` flow — the runner executes
the declared steps first, then invokes the deprivileged presentation body; direct Deno would not
receive the typed presentation input.

Run legacy TypeScript flows (no frontmatter `steps:`) with rote's bundled Deno from a directory
outside the active workspace — this keeps flow-created workspaces from nesting inside the
workspace you are using to inspect or author the flow:

```bash
rote deno run --allow-all /absolute/path/to/main.ts [args in declared order]
```

Do not use `rote run` as a fallback for normal TypeScript flow execution — stay with the
execution-model-appropriate command above. A flow with frontmatter `steps:` stays on
`rote flow run` even when tracking is requested; never route a DAG through `rote run`. If no
supported tracked wrapper exists, return that tracking limitation instead of changing runners.

Only an explicit legacy TypeScript flow with no frontmatter `steps:` may use `rote run` when the
scenario or command output requires model tracking or cached workspace responses. For that legacy
case, use this sequence:

```bash
rote init <workspace> --seq
cd ${ROTE_HOME:-$HOME/.rote}/rote/workspaces/<workspace>
rote model set <model> --provider <provider> --confirmed-current
rote run --inference-id $(uuidgen) \
  --model <model> \
  --model-type chat \
  --model-version <version> \
  /absolute/path/to/main.ts [args in declared order]
rote query @1 '.result' -r
```

Required tracking fields are `--inference-id`, `--model`, `--model-type`, and `--model-version`.

## Verification Criteria

- Verify the requested output artifact exists and contains the flow result.
- Check artifact content, not only file existence: requested path, key headings or markers, required
  parameter values, and live-data sections the user requested.
- Treat fully matched flow output as the answer. Do not overwrite, rewrite, reformat, enrich, or
  replace it unless the user explicitly asks for an edit or separate enhanced artifact.
- Existing released flows are already reusable. Running one unchanged does not create new reusable
  workflow knowledge and does not trigger `rote-flow-crystallization`.
- For partial matches, preserve the baseline flow as a reusable component for a composed superflow
  and route only the uncovered content onward.
- Do not treat a partial-flow output as the final augmented artifact. After uncovered work runs,
  verify the composed result contains both the baseline evidence and the new required capability.
- Preserve provenance literally for partial matches: flow name, parameters, output artifact,
  source labels, sentinels, markers, and any `FLOW_USED=...` or `source=...` text must survive as
  superflow source evidence. Do not paraphrase the baseline into a new hand-written report that
  erases flow evidence.
- For hybrid requests, record the baseline and the uncovered work before routing onward: baseline
  flow used, baseline output artifact, missing capability, selected adapter id(s), required
  probe/call responses, and final artifact markers that prove both parts are present. A successful
  baseline flow is not completion when the user asked for additional live/API data.

## Fallbacks

- If `rote flow info` is unavailable, fall back to `rote flow search "<intent>" --json` for path and
  parameter details.
- If JSON flow lookup is unavailable, resolve the flow from rote's flow listing and inspect only the
  flow frontmatter for parameters.
- Prefer upgrading rote or using live `rote grammar` guidance over filesystem searches.
- If execution is unsafe, parameters are missing, or the flow is only a partial match, stop flow
  execution and return the reason plus the preserved state.

## Return Fields

Return these fields to `rote` or the next skill:

- Flow name: registry or local flow name, if search reported it.
- Flow path: absolute path used for execution.
- Parameters: positional values and any unresolved required values.
- Execution command: exact command run or skip reason.
- Output artifact: path or cached response id.
- Verification result: what was checked and whether it satisfies the request.
- Coverage: full match, partial baseline, or skipped.
- Uncovered requirements: missing sources, capabilities, live observations, artifact sections, and
  verification checks.
- Preserved provenance: baseline flow name, parameters, output artifact, and any markers or source
  labels that must remain visible in the composed superflow.
- Next recommended skill: `rote-task-routing` for uncovered work, `rote-flow-crystallization` only
  for explicit new workflow/save work, or none for a verified full match.

## Handoff Contract

- Use when: a matched flow may satisfy all or part of the user request.
- Preconditions: `rote` has run the flow-search-first gate, or the user explicitly supplied a flow
  path and intent that can be validated against rote flow metadata.
- Owns: reading search JSON/frontmatter, resolving parameters, choosing execution mode, running the
  flow, preserving partial baseline output/provenance for superflow composition, and verifying
  user-visible results.
- Hands off to: `rote-task-routing` when uncovered work remains; `rote-flow-crystallization` only
  when the user requested new workflow/save work beyond unchanged flow reuse; `rote-troubleshooting`
  when unchanged retries keep failing.
- Returns to: `rote` with flow name, path, parameters, execution command, output artifact, coverage,
  and verification result.
- Stop when: the flow fully answers the request, required parameters are missing, execution would be
  unsafe, or the flow only establishes a baseline for another route. A verified full match returns
  no next skill.
- Completion signal: flow executed or skipped with reason, output verified or blocker named, and next
  recommended skill if any.
