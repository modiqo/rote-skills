---
name: rote-flow-crystallization
description: >
  Preserve reusable results from rote workspace, browser, or manual workflow work as pending flows.
  Use for pending write/save gates, save-or-discard decisions, and handoffs to flow authoring or
  registry sharing.
---

# rote-flow-crystallization

All `rote-<name>` references in this document — including every name in the Handoff
Contract — are companion **skills**, never CLI commands (`rote-shell` is not `rote shell`).
Invoke them through the runtime's skill mechanism; only literal `rote …` commands run in a
terminal.

Use this skill after workspace, browser, or manual execution produces a result that may be worth
saving as a reusable rote flow. The pending lifecycle is mandatory for reusable results: write the
pending stub first, run or record the pending save command second, then resolve save or discard.

A workflow is reusable when it has parameterizable inputs, repeatable adapter/browser/shell steps, a
stable output shape, and likely value for future agents. Do not treat a one-off user request as
non-reusable. If reusable or plausibly reusable, run this skill before final presentation: pending
write, pending save, then save/discard decision.

Do not use this skill after unchanged execution of an existing released flow that fully satisfied
the user request. That workflow is already reusable; verify and present the flow output instead.
Use this skill only when the run created new or changed workflow knowledge: adapter exploration,
browser steps, manual transformations, a composed superflow built around a partial-flow baseline, or
an explicit user request to save/release/publish a new workflow.

Do not use this skill for shell-only `rote proc` work unless `rote-shell` explicitly returns here.
Template and pending commands still require a real adapter, so process-only work uses the
adapterless workspace-export route instead of inventing one. That export is steps + presentation by default;
the explicit no-steps legacy TypeScript body is the escape only when the workflow needs control flow
or runtime interaction the step language does not support, or when the user asks for it — a set
discovered at run time is not a legacy trigger, since `for_each` resolves its width from an upstream
step's response (`rote-flow-authoring` owns the representability test, `rote grammar steps` the
syntax). An exported artifact is a draft — generalize recorded literals to `$param` and prune
inferred spurious parameters before lint (`rote-flow-authoring` owns that checklist). The
run-unchanged rule applies to the pending-save *scaffold command*, never to the exported artifact.
Mixed shell plus adapter, provider API, or browser workflows use this pending lifecycle. Only
workflows with no adapter, provider API, or browser state stay on the shell-only authoring route.

If the user already asked to save, release, publish, or make the workflow reusable, treat that as
save approval. Do not ask again, and do not skip `rote flow pending write` or `rote flow pending
save`.

## Pending Write Gate

Before presenting the final answer, create a pending flow stub that captures the repeatable steps,
inputs, adapter ids, cached response paths, and output shape from the completed run. Use live rote
guidance for exact syntax when uncertain: `rote grammar export` or `rote guidance agent essential`.

Precondition check before `pending write`:

- New workspace, browser, or manual steps produced the result; or
- A partial existing flow was combined with additional uncovered work; or
- The user explicitly asked to save, release, publish, or make a new workflow reusable.

If none of these are true and the result came from a verified full-flow match, return to the caller
with save gate `not applicable`.

Typical shape:

```bash
rote flow pending write <workspace-name> \
  --name <suggested-flow-name> \
  --adapter <adapter-id> \
  --description "<one-sentence purpose>" \
  --query '<validated jq query>' \
  --response-path '<validated jq path>' \
  --notes "<caveats, auth assumptions, data shape notes>"
```

Use repeated `--adapter` flags for multi-adapter work. The workspace name is positional; do not pass
`--workspace` to `pending write`, and do not pass template flags to `pending save`.
Pass only real adapters to `--adapter`; do not pass `process`, `shell`, or `adapter/process`.
Capture shell/process dependencies in notes, `deps.toml`, or TypeScript SDK shell calls.

The stub should preserve:

- The user's original intent.
- Any partial existing flow that was reused as a baseline, including its flow name, parameters,
  output artifact, source labels, and provenance that the composed superflow must preserve.
- The adapter calls, browser actions, or transformations that produced the result.
- Exact adapter ids selected for each capability. These ids are binding input to authoring unless
  the route is explicitly changed through `rote-task-routing`.
- Parameter candidates and values that must not be hard-coded.
- A concise result shape that a future flow can return.
- Auth, data-shape, and environment assumptions that affect reuse.

When crystallizing a partial-flow composition, the reusable unit is the new superflow, not a
one-off edited report. The stub should describe how to invoke or reproduce the baseline flow output
and how the uncovered work is merged around it. Do not replace the baseline with hand-written
content that merely resembles the earlier flow output.

For long-running work, interruption, or handoff, confirm the stub is recoverable:

```bash
rote flow pending list
```

If the session resumed after compaction or an interruption, also inspect workspace state before
scaffolding so the pending stub and cached evidence still point at the same route. These commands
resolve the active workspace from cwd — run them from inside the workspace directory
(`cd <workspace-path> && rote …`):

```bash
rote ls
rote workspace inspect meta
rote workspace inspect variables
rote flow pending list
```

## Pending Save Gate

After the stub exists, run `rote flow pending save` before answering the user:

```bash
rote flow pending save <workspace-name>
```

`pending save` prints the pre-filled `rote flow template create ...` command; it does not create or
release the flow. The pending artifact already encodes the default `steps:` plus presentation shape,
so capture the emitted command unchanged. Never append shape flags or reconstruct it after
compaction or handoff.

`pending save` only *prepares* this command; it does not order you to run it. Its output reads
"Scaffold command prepared. Run it only after the user has approved saving this as a reusable flow
(ask them if you haven't already)". So run the scaffold command yourself **only after** approval —
immediately if the user already approved, otherwise after the save question below. Scaffold output
is an agent action, never user-facing instructions.
When the pending state contains browser runtime or session data, run
`rote guidance browser flow-authoring` before editing the generated TypeScript body.

If the session was interrupted after writing the stub, recover with `rote flow pending list`, inspect
the relevant workspace name, and rerun `rote flow pending save <workspace-name>` before continuing
the save decision.

## Save Or Discard Decision

Only after pending write and pending save complete, present the task result and ask one explicit
yes/no question when the user did not already approve saving.

Use this shape:

```text
Result: <brief task result>

I can save this as a reusable rote flow for next time. Save it? (yes/no)
```

Do not combine the save question with unrelated follow-ups. Do not infer consent from silence,
thanks, or a new task.

If the user says yes or already asked to save, run the captured scaffold command and hand off to
`rote-flow-authoring` for implementation, tests, lint, release, index rebuild, search verification,
and optional registry sharing.

If the user says no, discard the pending stub through rote rather than deleting files directly:

```bash
rote flow pending discard <workspace-name>
```

If the answer is unclear, keep the pending stub and ask again for a yes/no decision. On resume, use
`rote flow pending list` to recover the stub and continue from the save question.

## Registry Handoff

When the saved flow should be shared, do not push directly from this skill. Return the pending stub,
captured scaffold command, target owner/namespace if known, and approval state to
`rote-flow-authoring`; after release, that skill hands off to `rote-registry`. A local release alone
has no published Play URI. When the downstream publication path returns a `play_uri`,
`install_command`, resolved run reference, published-reference `execution_verification` status and
evidence, and visibility-based sharing guidance, present and propagate them instead of constructing
or parsing the URI here. Also propagate execution readiness and blockers; a published installer URI
is not necessarily executable by `play run`, and static eligibility is not proof of successful
execution.

## Return Fields

Return these fields to `rote`, `rote-workspace`, `rote-browse`, or `rote-flow-authoring`:

- Workspace name: source workspace or browser capture context.
- Pending stub: name, adapters, query/response path, and notes.
- Pending save command: exact emitted scaffold command or recovery command.
- Save decision: accepted, declined, unclear, or pre-approved by the original request.
- Release recommendation: draft only, local release, publish/share, or no release.
- Published Play URI, execution readiness, blockers, published-reference execution-verification
  status and evidence, and visibility-based access guidance returned by the downstream registry path
  when published; otherwise none.
- Next recommended skill: `rote-flow-authoring`, `rote-registry`, `rote-troubleshooting`, or none.

## Handoff Contract

- Use when: completed rote, browser, or manual work produced a result that may be reusable.
- Preconditions: an owning skill has a user-visible result plus workspace, browser, or command state
  sufficient to describe repeatable inputs and outputs.
- Owns: pending write, pending save, save/discard decision handling, stub recovery, and transfer of
  approved reusable work to authoring.
- Hands off to: `rote-flow-authoring` after save approval; `rote-registry` only after an already
  released artifact needs sharing; `rote-troubleshooting` when pending commands fail repeatedly.
- Returns to: `rote`, `rote-workspace`, `rote-browse`, or the delegating skill with pending state,
  decision, scaffold command, verified Play URI and sharing guidance when published, and next owner.
- Stop when: the user declines saving, the save decision is unclear and needs input, the pending stub
  is unrecoverable, or authoring becomes the correct owner.
- Completion signal: pending stub saved or discarded, decision recorded, and next skill or final
  answer named.
