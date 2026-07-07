---
name: rote-flow-authoring
description: >
  Create, edit, test, lint, release, verify, or publish reusable rote flows. Use for reusable
  contract elicitation, rote-driven schema discovery, scaffold/test/release/search lifecycle, and
  registry handoff.
---

# rote-flow-authoring

All `rote-<name>` references in this document — including every name in the Handoff
Contract — are companion **skills**, never CLI commands (`rote-shell` is not `rote shell`).
Invoke them through the runtime's skill mechanism; only literal `rote …` commands run in a
terminal.

Use this skill when the user asks to create, edit, lint, release, verify, or publish a reusable rote
flow, or when `rote-flow-crystallization` returns an approved pending save command. Keep live command
help authoritative for syntax: `rote grammar export`, `rote grammar deno`, `rote guidance typescript
essential`, and `rote guidance registry essential`.

## Elicit The Reusable Contract

Before scaffolding, identify the repeatable workflow boundary:

- User-visible goal and success condition.
- Required inputs, optional inputs, and safe defaults.
- Whether the flow is atomic or a composed superflow that reuses an existing partial-flow baseline.
- Adapter operations, browser actions, or shell/process operations needed to produce the result.
- Which captured response, file, or snapshot will back each required source and capability.
- Output shape a future caller should receive.
- Values that must remain parameters rather than hard-coded secrets, dates, IDs, or names.

For a composed superflow, keep the baseline flow as a first-class reusable dependency in the
contract. Record the baseline flow name, parameters, output artifact or structured output, and the
merge rule for uncovered work. Do not hard-code the baseline's last report contents into the new
flow body unless the user explicitly asked for a static snapshot. The release test must prove the
output contains baseline evidence plus each newly required capability.

If the requirement depends on an API shape, discover it through rote adapter probes and calls in a
workspace before writing the flow.

Do not replace adapter-backed discovery with direct HTTP from `curl`, Python, Node, raw `fetch`, or
provider SDKs. If no installed adapter fits, return to `rote-task-routing` for catalog search and
installation before authoring.

## Discover Input Schemas Through Rote

Use rote commands to inspect adapter capabilities and response examples. Prefer cached response IDs
and jq filters over copying large JSON into the prompt.

For each candidate parameter, record:

| Parameter | Required? | Source | Notes |
| --- | --- | --- | --- |
| User input | yes/no | Prompt, adapter schema, or default | Include validation or accepted shape. |

Hard rules for reusable adapter parameters:

- Probe first, then read the selected tool's input schema before choosing flags.
- For every server-side filter or input dimension, expose a CLI flag, hard-code it with a short
  reason, or omit it with a short reason.
- If the API accepts a structured filter, query, body, or params object, include a raw JSON
  passthrough flag such as `--filter` or `--body` unless live guidance says not to.
- Keep frontmatter parameters in lockstep with CLI flags: same names, types, defaults, required
  state, and descriptions that mention the underlying API field.
- Test different server-side dimensions, not only the same pagination or output knob with different
  values.

## Choose The Flow Shape

Before scaffolding or exporting, classify the workflow from the user request, existing flow
frontmatter, and workspace evidence. Use live `rote grammar export` and
`rote flow template create --help` to confirm current command flags.

This classification chooses the flow runner shape. It does not bless a schema migration or make a
new adapter contract scheme the default.

| Authoring target | Choose when | Artifact marker | Create/export path |
| --- | --- | --- | --- |
| Legacy TypeScript, no `steps:` | The workflow is inherently imperative or depends on dynamic SDK/browser/local orchestration. | No frontmatter `steps:`; effects live in the TypeScript body. | `rote flow template create ...`, then `rote deno run --allow-all` |
| `steps:` DAG | Adapter/process calls are the reusable execution plan and the default DAG report/checkpoints are enough. | Frontmatter `steps:`; effects live in declarative steps. | Use the current `--with-steps` template/export path advertised by live rote help. |
| `steps_with_presentation` | Effects are declarative, but callers need custom human, summary, or JSON rendering. This is a presentation variant of a `steps:` flow. | `steps:` for effects; `metadata.execution_model: steps_with_presentation`; TypeScript body is presentation-only. | Use the current `--with-steps --with-presentation` template/export path advertised by live rote help. |

Do not put adapter calls, `Rote`, `runPreflight`, `fetch`, `Deno.Command`, direct env reads, or
filesystem access in a `steps_with_presentation` body. That body reads completed step outputs
through `__ROTE_PRESENTATION_SDK__`.

## Scaffold Through Rote

Use the pending save command from `rote-flow-crystallization` when authoring follows a completed
task. This applies even when the original request already said to save or release the flow: pending
write and pending save come first, then the emitted scaffold command.

The emitted scaffold command creates the legacy body shape. When the flow-shape choice above calls
for a `steps:` flow, append `--with-steps` (and `--with-presentation` when custom rendering is
needed) to the emitted `rote flow template create` command before running it.

For adapter-backed or mixed adapter/shell work, "save this" means preserve workspace history, run
pending write/save, and scaffold from rote with `rote flow template create`. It does not authorize
creating `~/.rote/flows/<name>` by hand or transcribing the last successful script into a flow.

For direct authoring, use the current scaffold/export syntax from `rote grammar export`.
`rote flow template create` is flag-driven: pass `--name <flow-name>`, repeated
`--adapter <adapter-id-or-endpoint>` flags, and `--workspace <workspace-name>` when scaffolding from
workspace history. Do not create flow directories by hand unless live rote guidance says to.

Preserve adapter ids from the pending stub and the routing decision. Do not swap to an adjacent
provider adapter during scaffold or implementation because the new id appears cleaner, newer, or
easier to call. If the selected adapter cannot satisfy the capability, stop authoring and return to
`rote-task-routing` with the missing capability.

For shell-only flows handed off by `rote-shell`, follow the shell crystallization discipline instead
of forcing an adapter-shaped scaffold: author `~/.rote/flows/<name>/main.ts` with
`@rote-frontmatter`, create `deps.toml`, use `rote.exec`/shell SDK calls, run dependency preflight,
and test with `rote deno run --allow-all`. Do not run `rote flow pending save`,
`rote flow template create`, or `rote flow frontmatter` for a shell-only flow unless live shell
guidance says those commands now support the required process-only shape.

## Flow Runtime Boundary

Flow code already runs under `rote deno run`; do not recreate rote lifecycle inside it.

- Do not run `rote init` from inside a TypeScript flow.
- Do not shell out to `rote init` as a workaround for `Rote.workspace(...)` failures.
- Do not add `rote proc run` calls to a reusable flow body.
- Workspace bootstrap belongs outside flow execution unless the SDK owns it.

If `Rote.workspace(...)` creates a nonexistent cwd or fails, treat it as SDK/runtime failure.
Inspect `rote deno status`, `rote sdk status`, and `rote guidance typescript flow-creation`.

## Implement The Flow Body

For TypeScript flow logic, hand off to `rote-typescript-transformations` or follow its rules before
continuing this lifecycle.

For flows whose frontmatter declares `metadata.execution_model: steps_with_presentation`, the
TypeScript body is presentation-only. Import `__ROTE_PRESENTATION_SDK__`, read completed outputs
with `loadPresentationContext()` and literal `stepName("...")` references, and render with
`FlowOutput`. Do not import the broad SDK, construct `Rote`, run preflight, create a task queue,
call `fetch`, spawn subprocesses, or read `Deno.args`/`Deno.env` directly from that body.

Preserve a stable `FlowOutput` shape:

- Return structured data for machine reuse.
- Include a concise human-readable summary when useful.
- For superflows, include enough structure to distinguish baseline-flow output from newly added
  adapter/browser/process data.
- Keep adapter raw responses out of the final result unless the user needs them.
- Avoid embedding local workspace paths, secrets, or one-off IDs in output.

When the implementation needs local shell processing, use `rote proc` only during exploration
outside the reusable flow. Inside the flow body, use TypeScript shell SDK wrappers (`rote.exec`,
`rote.execBackground`, `rote.followFile`, and related helpers) or declarative `process.exec` steps.
Do not add raw child-process code, shell pipelines, inline script snippets, or `rote proc run`.

When the implementation needs provider/API data, use the adapter calls and cached responses from the
scaffolded workspace history. Do not add raw HTTP fetches to a flow body to bypass missing adapter
setup; fix the adapter route or hand back to `rote-task-routing`.

If `rote deno run` fails because the generated SDK import cannot find `mod.ts`, `Adapter.callBg` is
undefined, or the runtime cannot find rote-managed Deno/SDK state, treat that as a scaffold/runtime
issue. Inspect `rote deno status`, `rote sdk status`, `rote grammar deno`, and
`rote guidance typescript flow-creation`; do not keep hand-editing around the error with raw Deno,
raw HTTP, or unrelated SDK APIs.

## Test, Lint, Release, And Search

Run legacy TypeScript flows, with no frontmatter `steps:` block, from outside the active workspace
with representative parameter sets:

```bash
rote deno run --allow-all /absolute/path/to/main.ts [args]
```

Run flows with frontmatter `steps:` through the flow runner instead. For presentation flows, it
executes effects first and then invokes the deprivileged presentation body:

```bash
rote flow run /absolute/path/to/main.ts param=value
```

For shell flows, run the shell entrypoint directly from outside the active workspace. Cover the
common case, empty or no-result case, optional defaults, and at least one user-provided edge case.

Release/lint success is not capability proof. Before release, rerun the capability check:
every required source is backed by a captured response, every adapter id from the routing decision
was installed and called, and the output contains baseline evidence plus each newly required
capability. Do not continue to release if lint/runtime succeeds while a required capability is
still missing.

Before calling the work complete, run the live lint/release path surfaced by rote. Release is a hard
gate, not a file edit. Execution success and `rote flow validate` do not make a flow discoverable;
only `rote flow release <name>` performs the local lifecycle transition.
Never use `rote flow release --force` to satisfy a save/release task. A forced release is a broken
artifact state for humans to inspect, not an agent completion path.

Before release, obtain explicit authorization unless the original request already asked to release,
crystallize, finalize, make discoverable, save as reusable, or publish the flow. After release,
verify discoverability:

```bash
rote flow lint <name>
rote flow release <name>
rote flow index --rebuild
rote flow search "<intent-or-flow-name>"
rote flow search "<intent-or-flow-name>" --json
```

Before pending cleanup, verify the smoke run's final artifact content, including baseline markers
and newly required live/API sections.

If lint or release fails, change the flow, arguments, adapter configuration, cwd, or environment
before retrying. Do not edit frontmatter `status` by hand.

If this authoring run came from a pending stub, cleanup is part of release completion:

```bash
rote flow pending discard <workspace-name>
```

Run pending discard only after release succeeded, the index was rebuilt, and search verification
shows the released flow. Do not leave `.pending-flow.json` or a workspace-scoped pending stub behind
as a resume anchor after the released flow is discoverable.

## Registry Handoff

If the flow should be shared, use `rote guidance registry essential` and `rote grammar registry` for
the current push syntax. Confirm the target namespace before publishing, then hand off to
`rote-registry` with flow path, release status, owner/namespace, visibility, dependency notes, and
the user's publish approval.

## Return Fields

Return these fields to `rote`, `rote-flow-crystallization`, or `rote-registry`:

- Flow name and path.
- Parameter contract and schema decisions.
- Test commands and representative inputs.
- Lint, release, index, search verification, and pending cleanup output.
- Registry target, visibility, and publish approval if sharing is requested.
- Next recommended skill: `rote-typescript-transformations`, `rote-registry`, `rote-troubleshooting`,
  or none.

## Handoff Contract

- Use when: a reusable flow must be created, edited, tested, linted, released, verified, or prepared
  for publication.
- Preconditions: user intent or a pending save command defines the reusable workflow boundary; any
  required API shape can be discovered through rote before scaffolding.
- Owns: contract elicitation, schema discovery, scaffold, implementation lifecycle, tests, lint,
  release, index/search verification, pending cleanup when applicable, and registry-ready return
  data.
- Hands off to: `rote-typescript-transformations` for TypeScript flow logic; `rote-registry` for
  sharing; `rote-flow-run` for final execution verification; `rote-troubleshooting` after repeated
  unchanged failures.
- Returns to: `rote` or `rote-flow-crystallization` with flow path, parameter contract, verification
  status, release state, and next owner.
- Stop when: the flow is verified, a release/publish approval is needed, a required schema or
  credential is missing, or troubleshooting becomes the correct owner.
- Completion signal: flow draft, release plus index/search verification and pending cleanup when
  applicable, publish handoff, or blocker is named with commands already run.
