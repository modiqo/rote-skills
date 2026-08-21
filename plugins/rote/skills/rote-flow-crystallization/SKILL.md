---
name: rote-flow-crystallization
description: >
  Distill reusable workspace, browser, shell, or manual evidence into a semantic play plan. Owns
  save-or-discard, pending write/save where supported, and handoff to play authoring.
---

# rote-flow-crystallization

All `rote-<name>` references in this document — including every name in the Handoff
Contract — are companion **skills**, never CLI commands (`rote-shell` is not `rote shell`).
Invoke them through the runtime's skill mechanism; only literal `rote …` commands run in a
terminal.

Use this skill after workspace, browser, shell, or manual execution produces a result that may be
worth saving as a reusable rote play. This skill owns the transition from evidence to a reusable
procedure. Adapter, browser, and mixed workflows persist that decision through pending write/save;
process-only workflows use the same planning and approval gate before adapterless workspace export.

A workflow is reusable when it has parameterizable inputs, repeatable adapter/browser/shell steps, a
stable output shape, and likely value for future agents. Do not treat a one-off user request as
non-reusable. If reusable or plausibly reusable, run this skill before final presentation: build the
crystallization plan, persist it through pending write/save when supported, then resolve save or
discard.

Do not use this skill after unchanged execution of an existing released play that fully satisfied
the user request. That workflow is already reusable; verify and present the play output instead.
Use this skill only when the run created new or changed workflow knowledge: adapter exploration,
browser steps, manual transformations, a composed superplay built around a partial-play baseline, or
an explicit user request to save/release/publish a new workflow.

Process-only `rote proc` work enters this skill when `rote-shell` identifies a reusable result.
Template and pending commands still require a real adapter, so process-only work skips those commands
without skipping crystallization: build the plan below, resolve save approval, then hand the approved
plan to `rote-flow-authoring` for adapterless workspace export. Never invent an adapter. Mixed shell
plus adapter, provider API, or browser workflows use pending write/save normally.

If the user already asked to save, release, publish, or make the workflow reusable, treat that as
save approval. Do not ask again. For pending-capable workflows, do not skip `rote play pending write`
or `rote play pending save`.

## Build The Crystallization Plan

Use `rote guidance play crystallization` as the canonical plan contract; produce every field it
defines without redefining them here. Record the plan in pending notes when the workflow supports
pending write; carry it directly to authoring for process-only export. Read `rote guidance play
shape` when mapping the plan into a DAG and `rote guidance play testing` before release. Exact step
syntax belongs to `rote grammar steps`.

## Pending Write Gate

For adapter, browser, and mixed workflows, create a pending play stub before presenting the final
answer. It captures the crystallization plan, inputs, adapter ids, cached response paths, and output
shape from the completed run. Process-only workflows skip to the save decision after producing the
plan. Use live rote guidance for exact pending syntax when uncertain: `rote grammar export`.

Precondition check before `pending write`:

- New workspace, browser, or manual steps produced the result; or
- A partial existing play was combined with additional uncovered work; or
- The user explicitly asked to save, release, publish, or make a new workflow reusable.

If none of these are true and the result came from a verified full-play match, return to the caller
with save gate `not applicable`.

Typical shape:

```bash
rote play pending write <workspace-name> \
  --name <suggested-play-name> \
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
- Any partial existing play that was reused as a baseline, including its play name, parameters,
  output artifact, source labels, and provenance that the composed superplay must preserve.
- The adapter calls, browser actions, or transformations that produced the result.
- Exact adapter ids selected for each capability. These ids are binding input to authoring unless
  the route is explicitly changed through `rote-task-routing`.
- Parameter candidates and values that must not be hard-coded.
- A concise result shape that a future play can return.
- Auth, data-shape, and environment assumptions that affect reuse.
- Package inputs and address positions needed by the reusable play. Record the facts without
  deciding their shipped layout here; `rote-flow-authoring` applies the authoritative contract from
  `rote guidance typescript play-creation`.

When crystallizing a partial-play composition, the reusable unit is the new superplay, not a
one-off edited report. The stub should describe how to invoke or reproduce the baseline play output
and how the uncovered work is merged around it. Do not replace the baseline with hand-written
content that merely resembles the earlier play output.

For long-running work, interruption, or handoff, confirm the stub is recoverable:

```bash
rote play pending list
```

If the session resumed after compaction or an interruption, also inspect workspace state before
scaffolding so the pending stub and cached evidence still point at the same route. These commands
resolve the active workspace from cwd — run them from inside the workspace directory
(`cd <workspace-path> && rote …`):

```bash
rote ls
rote workspace inspect meta
rote workspace inspect variables
rote play pending list
```

## Pending Save Gate

After the stub exists, run `rote play pending save` before answering the user:

```bash
rote play pending save <workspace-name>
```

`pending save` prints the pre-filled `rote play template create ...` command; it does not create or
release the play. The pending artifact already encodes the default `steps:` plus presentation shape,
so capture the emitted command unchanged. Never append shape flags or reconstruct it after
compaction or handoff.

`pending save` only *prepares* this command; it does not order you to run it. Its output reads
"Scaffold command prepared. Resolve the save decision first; after acceptance or pre-approval, run
`rote guidance play shape` and execute this command unchanged". Preserve the emitted command
unchanged in the handoff; never run it from crystallization, present it as user instructions, append
shape flags, or reconstruct it from memory.

If the session was interrupted after writing the stub, recover with `rote play pending list`, inspect
the relevant workspace name, and rerun `rote play pending save <workspace-name>` before continuing
the save decision.

### Crystallization Value

`pending save` also reports a typed crystallization value under `@@share` (`data.share_nudge` under
`--json`). Read those fields; do not restate them from memory or compute your own. Its readiness is
`not_assessed` deliberately: no artifact exists yet, so nothing about publication has been checked.
Saving is what creates the flow that `rote play lint` and `rote play release` then validate. Never
describe a pending flow as publishable, shareable, or ready to publish.

## Save Or Discard Decision

For pending-capable workflows, ask only after pending write and pending save complete. For
process-only workflows, ask after the crystallization plan is complete. Present the task result and
ask one explicit yes/no question when the user did not already approve saving.

Use this shape:

```text
Result: <brief task result>

I can save this as a reusable rote play for next time. Save it? (yes/no)
```

Do not combine the save question with unrelated follow-ups. Do not infer consent from silence,
thanks, or a new task.

If the user says yes or already asked to save, hand off the crystallization plan and exact pending
scaffold command to `rote-flow-authoring` **without executing it**. Authoring runs the pending
scaffold unchanged for pending-capable workflows; for process-only workflows, it materializes the
approved plan through adapterless workspace export. Authoring then owns implementation, tests, lint,
release, index rebuild, search verification, and optional registry sharing.

If the user says no, discard a pending-capable workflow's stub through rote rather than deleting
files directly:

```bash
rote play pending discard <workspace-name>
```

For process-only work, record the declined decision and leave the captured evidence unchanged; there
is no pending stub to discard. If the answer is unclear, retain any pending stub and the plan, then
ask again for a yes/no decision. On resume, use `rote play pending list` when applicable.

## Registry Handoff

When the saved play should be shared, do not push directly from this skill. Return the pending stub,
captured scaffold command, target owner/namespace if known, and approval state to
`rote-flow-authoring`; after release, that skill hands off to `rote-registry`. A local release alone
has no published Play URI. When the downstream publication path returns a `play_uri`,
`bootstrap_uri`, resolved run reference, published-reference `execution_verification` status and
evidence, and access guidance (resolution and execution audiences), present and propagate them instead of constructing
or parsing the URI here. Also propagate execution readiness and blockers; a published installer URI
is an advertised transition rather than the Play identity, and static eligibility is not proof of
successful execution.

## Return Fields

Return these fields to `rote`, `rote-workspace`, `rote-browse`, or `rote-flow-authoring`:

- Workspace name: source workspace, browser capture context, or process workspace.
- Crystallization plan: the eight-field contract defined by `rote guidance play crystallization`.
- Pending stub: name, adapters, query/response path, and notes, or none for process-only work.
- Pending save command: exact emitted scaffold command or recovery command, not executed by this
  skill; none for process-only work.
- Save decision: accepted, declined, unclear, or pre-approved by the original request.
- Release recommendation: draft only, local release, publish/share, or no release.
- Exploration cost evidence: the two token bases separately, or none. Never their sum — the context
  figure is a heuristic that may overlap the counted payload, so a total is a number neither source
  supports. Relaying nothing is better than relaying one base as if it were both.
- Published Play URI, execution readiness, blockers, published-reference execution-verification
  status and evidence, and access guidance (resolution and execution audiences) returned by the downstream registry path
  when published; otherwise none.
- Next recommended skill: `rote-flow-authoring`, `rote-registry`, `rote-troubleshooting`, or none.

## Handoff Contract

- Use when: completed rote, browser, shell, or manual work produced a result that may be reusable.
- Preconditions: an owning skill has a user-visible result plus workspace, browser, or command state
  sufficient to describe repeatable inputs and outputs.
- Owns: the crystallization plan, pending write/save where supported, save/discard decision handling,
  stub recovery, and transfer of approved reusable work to authoring.
- Hands off to: `rote-flow-authoring` after save approval; `rote-registry` only after an already
  released artifact needs sharing; `rote-troubleshooting` when pending commands fail repeatedly.
- Returns to: `rote`, `rote-workspace`, `rote-browse`, or the delegating skill with pending state,
  decision, scaffold command, the published-play fields listed above when published (URI, execution
  readiness, blockers, execution-verification status and evidence, access guidance), and next owner.
- Stop when: the user declines saving, the save decision is unclear and needs input, the pending stub
  is unrecoverable, or authoring becomes the correct owner.
- Completion signal: crystallization plan returned, supported pending state saved or discarded,
  decision recorded, and next skill or final answer named. No scaffold or export command has been
  executed by this skill.
