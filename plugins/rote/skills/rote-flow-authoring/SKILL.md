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
help authoritative for syntax.

## Route The Implementation

Choose the implementation guide before editing the scaffold:

| Workflow concern | Required guidance |
| --- | --- |
| Browser navigation, snapshots, clicks, typing, browser auth, or replay | `rote guidance browser flow-authoring` |
| Cached-response transformation or general TypeScript logic | `rote-typescript-transformations` plus `rote grammar deno` |
| Shell/process execution | The shell authoring route from `rote-shell` |
| Registry publication | `rote guidance registry essential` |

Browser TypeScript does not belong to the generic transformation route. Run `rote guidance browser
flow-authoring`, follow its typed-step, presentation, and legacy-SDK procedure, then return here for tests, lint, release,
index/search verification, and pending cleanup.

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

Reusable flows use steps + presentation by default. The effect plane is the frontmatter `steps:` DAG; the
presentation plane is the deprivileged TypeScript body selected by
`metadata.execution_model: steps_with_presentation`. Adapter, `process.exec`, and `browser.*`
effects all belong in `steps:`. The body only normalizes and renders the completed observations.

Use `rote grammar steps` for the step-syntax quick reference, the worked example in
`rote guidance typescript flow-creation` for the complete assembled artifact, and live
`rote grammar export` plus `rote flow template create --help` for exact scaffold syntax. Choose an
explicit escape from the default only when the contract calls for it.

The shape test is representability, not a fixed list of blessed reasons. Prefer typed steps whenever
the workflow fits a finite DAG of dependencies, conditions, fan-out, concurrency, and finite process
budgets. Reach for a legacy no-steps body only when the workflow needs control flow or runtime
interaction the current step language does not support, or when the user explicitly asks for one.
PTY interaction and background leases/streams with concurrent mid-lease work are the known examples,
not the whole set — adaptive pagination to exhaustion, recursive traversal, data-dependent loop
termination, runtime tool selection, and stateful session protocols are gaps in the step language
too. A set whose width is only discovered at run time is *not* one of them: `for_each` resolves
width from the upstream step's response at run time. When you do escape, name the specific
capability the step language is missing; if you cannot name one, the workflow is representable.

| Authoring target | Choose when | Artifact marker | Create/export path |
| --- | --- | --- | --- |
| Default steps + presentation flow | The reusable workflow has adapter, process, browser, or mixed effects and needs stable human/summary/JSON output. | Frontmatter `steps:` AND `metadata.execution_model: steps_with_presentation`; the TypeScript body is presentation-only. | Use template/export with no shape flags, or run the command emitted by pending save unchanged. |
| Explicit steps-only DAG | The typed effect plan and runner report are the entire contract; no custom TypeScript rendering is wanted. | Frontmatter `steps:` WITHOUT `metadata.execution_model: steps_with_presentation`. | Request `--with-steps` for a template or `--format steps` for export, without a presentation flag. |
| Explicit legacy TypeScript, no `steps:` | The workflow needs control flow or runtime interaction the step language does not support (name it — see the representability test above), or the user explicitly requested a plain no-steps body. | No frontmatter `steps:`; effects live in the TypeScript body. | Request `--legacy-body` where live help advertises it — `template create` still requires `--adapter`, so an adapterless shell-only flow has no generator: hand-author it from the legacy example in the rote-shell skill or `rote guidance typescript flow-creation`; run with `rote deno run --allow-all`. |

These shape defaults do not migrate the adapter contract scheme. Schema v1 remains the default;
schema v2 is an explicit `--scheme 2` opt-in and must satisfy its own live contract/origin rules.

Do not put adapter calls, `Rote`, `runPreflight`, `fetch`, `Deno.Command`, direct env reads, or
filesystem access in a `steps_with_presentation` body. That body reads completed step outputs
through `__ROTE_PRESENTATION_SDK__`.

The effect plane, minimally — an adapter step, a local-command step, and cross-step data flow.
Remaining `parameters:` entries are elided for brevity: declare EVERY `$param` a step references
(an unresolved token passes through as a literal, not an error):

```yaml
parameters:            # top level, not under metadata:
- name: owner
  type: string
  required: true
  description: "GitHub repository owner"
# …declares other parameters (root, repo, title, head) in a similar way
steps:
  changelog:                          # local command: typed process.exec, argv list
    type: process.exec
    argv: [git, -C, $root, log, --oneline, -n, "20"]
  create_pr:                          # adapter call: endpoint + method + params
    endpoint: adapter/github
    method: pulls/create
    params: { owner: $owner, repo: $repo, title: $title, head: $head, base: main }
  verify_pr:                          # adapter call: endpoint + method + params, consumes create_pr's unwrapped body
    endpoint: adapter/github
    method: pulls/get
    depends_on: [create_pr]
    params: { owner: $owner, repo: $repo, pull_number: "@create_pr{.number}" }
```

`$param` substitutes into every step string field — adapter `params:` values and `process.exec`
`argv` elements alike; an unresolved token passes through as a literal, not an error. `@step{.path}`
reads a completed step's unwrapped body (never `.result.content[0].text`); `for_each: '$.items'`
fans a step out over the source step's array with `$item`/`$item_index` bound per element. Fan-out
width is resolved at run time, so a set discovered by an upstream step is NOT a legacy trigger —
over a process step's JSON stdout use `for_each: '$.stdout.text | fromjson'` (bridge example in
`rote guidance typescript flow-creation`). A `process.exec` step takes an optional `timeout_ms:`
budget (default 30s). In the presentation body, flow parameters arrive on `ctx.params` — never
echo them through a step's stdout and never read `Deno.args`.

## Scaffold Through Rote

Use the pending save command from `rote-flow-crystallization` when authoring follows a completed
task. This applies even when the original request already said to save, release, publish, or make the
workflow reusable: pending write and pending save come first, then the emitted scaffold command.
`pending save` only prepares that command — run it only after the user has approved saving
(immediately when authoring follows an already-approved handoff, otherwise after the save question).
Approval is the single gate both this skill and `rote-flow-crystallization` share.

The emitted scaffold command already encodes the pending artifact's steps + presentation shape. Run it
unchanged. Never append shape flags, rebuild it from memory, or replace it with a second scaffold
command; doing so can discard the recorded adapters, browser sessions, workspace, or execution
model.

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

For process-only work handed off by `rote-shell`, do not invent an adapter just to satisfy template
or pending commands: those adapterless paths remain unsupported. Prefer the no-shape-flag workspace
export over the recorded `rote proc` trace; it emits `process.exec` steps plus presentation. If the
representability test above lands on a legacy body — a named gap in the step language, or an
explicit user request — author
`~/.rote/flows/<name>/main.ts` with `@rote-frontmatter`, create `deps.toml`, use the shell SDK, run
dependency preflight, and test that legacy body with `rote deno run --allow-all`.

A workspace export is a draft synthesized from one recording, not a finished flow. `rote proc run`
records literal argv, so the export carries recorded literals (paths, owners, dates) and may lift
recorded values into spurious `parameters:` entries. Before lint: replace recorded literals with
`$param` tokens, declare each real parameter, and prune every inferred parameter that is not part
of the contract. Generalizing parameters and the presentation body is expected; changing the shape
by hand (adding/removing `steps:` or `execution_model`) is not — re-export with the explicit flag
instead. The "run the emitted command unchanged" rule protects the pending-save *scaffold command*,
not the exported artifact.

## Flow Runtime Boundary

Flow code already runs under its owning runner (`rote flow run` for any `steps:` flow, bundled Deno
only for an explicit no-steps legacy body); do not recreate rote lifecycle inside it.

- Do not run `rote init` from inside a TypeScript flow.
- Do not shell out to `rote init` as a workaround for `Rote.workspace(...)` failures.
- Do not add `rote proc run` calls to a reusable flow body.
- Workspace bootstrap belongs outside flow execution unless the SDK owns it.

If `Rote.workspace(...)` creates a nonexistent cwd or fails, treat it as SDK/runtime failure.
Inspect `rote deno status`, `rote sdk status`, and `rote guidance typescript flow-creation`.

## Implement The Flow Body

For browser TypeScript, run `rote guidance browser flow-authoring`. For non-browser TypeScript logic,
hand off to `rote-typescript-transformations` or follow its rules before continuing this lifecycle.

For flows whose frontmatter declares `metadata.execution_model: steps_with_presentation`, the
TypeScript body is presentation-only. Import `__ROTE_PRESENTATION_SDK__`, read completed outputs
with `loadPresentationContext()` and literal `stepName("...")` references, and render with
`FlowOutput`. Read the flow's invocation parameters from `ctx.params`
(`Readonly<Record<string, unknown>>`, declared defaults already applied) — that is the sanctioned
input surface. Do not import the broad SDK, construct `Rote`, run preflight, create a task queue,
call `fetch`, spawn subprocesses, or read `Deno.args`/`Deno.env` directly from that body.

Live progress belongs to the runner because the presentation body starts only after declared steps
finish. A request for a spinner or staged progress must not move adapter calls into TypeScript,
remove `steps:`, or switch the flow to a legacy body. Preserve the DAG and use a runner-owned
progress surface when available; otherwise state that live presentation progress is not supported
for this execution model instead of publishing a stepless substitute.

Normalize each observation without assuming a provider schema. Preserve `single`, `fan_out`, and
`empty_fan_out`; use `requireAvailable` for completed/checkpoint-restored output and branch on the
outcome for failed, skipped, or blocked steps. Narrow process and browser bodies with
`isProcessExecBody` and `isBrowserBody`. Treat remaining adapter bodies as direct JSON, plain text,
or narrowly recognized legacy text-envelope residue; malformed/optional values stay explicit and
must not become fabricated empty strings, zeroes, false values, or arrays. Use the concrete pattern
in `rote guidance typescript flow-creation`.

Preserve a stable `FlowOutput` shape:

- Return structured data for machine reuse.
- Include a concise human-readable summary when useful.
- For superflows, include enough structure to distinguish baseline-flow output from newly added
  adapter/browser/process data.
- Keep adapter raw responses out of the final result unless the user needs them.
- Avoid embedding local workspace paths, secrets, or one-off IDs in output.

When the implementation needs local shell processing, use `rote proc` only during exploration.
Default steps + presentation flows put finite work in declarative `process.exec` steps; their presentation body
never runs shell SDK calls. Only an explicit no-steps legacy body uses `rote.exec`,
`rote.execBackground`, `rote.followFile`, and related wrappers. Never add raw child-process code,
shell pipelines, inline script snippets, or `rote proc run` to either body.

When the implementation needs provider/API data, default steps + presentation flows put adapter endpoint/method
calls in frontmatter `steps:` and presentation reads their typed observations. Only an explicit
legacy body uses adapter handles returned by `runPreflight`. Do not add raw HTTP fetches to bypass
missing adapter setup; fix the adapter route or hand back to `rote-task-routing`.

If an explicit legacy body's `rote deno run` fails because the generated SDK import cannot find
`mod.ts`, `Adapter.callBg` is undefined, or the runtime cannot find rote-managed Deno/SDK state,
treat that as a scaffold/runtime issue. Inspect `rote deno status`, `rote sdk status`, `rote grammar
deno`, and `rote guidance typescript flow-creation`; do not hand-edit around it with raw Deno, raw
HTTP, or unrelated SDK APIs.

## Test, Lint, Release, And Search

Run flows with frontmatter `steps:` (the default shape) through the flow runner, from outside the
active workspace, with representative parameter sets. For presentation flows, it executes effects
first and then invokes the deprivileged presentation body:

```bash
rote flow run /absolute/path/to/main.ts param=value
```

Run an explicit legacy TypeScript flow, with no frontmatter `steps:` block, through bundled Deno
instead:

```bash
rote deno run --allow-all /absolute/path/to/main.ts [args]
```

The DAG runner takes named `key=value` parameters; the legacy body takes its declared positional
argument order.

Whichever runner applies, cover the common case, empty or no-result case, optional defaults, and
at least one user-provided edge case.

Release/lint success is not capability proof. Before release, rerun the capability check:
every required source is backed by a captured response, every adapter id from the routing decision
was installed and called, and the output contains baseline evidence plus each newly required
capability. Do not continue to release if lint/runtime succeeds while a required capability is
still missing.

Before calling the work complete, run the live lint/release path surfaced by rote. Release is a hard
gate, not a file edit. Execution success and `rote flow validate` do not make a flow discoverable;
only `rote flow release <name>` performs the local lifecycle transition.
Never use `rote flow release --force` to satisfy a save/release/publish task. A forced release is a
broken artifact state for humans to inspect, not an agent completion path.

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
the user's publish approval. A local release alone has no published Play URI. When `rote-registry`
returns a `play_uri`, `install_command`, resolved run reference, published-reference
`execution_verification` status and evidence, and visibility-based sharing guidance after
publication or an already-in-sync check, present and propagate them; do not construct or parse the
URI in this skill. Treat the Play URI, static execution readiness, and successful execution as
separate facts: propagate `play_run_eligible`, the execution variant, blockers, and verification
status, and do not describe an unverified or failed version as successfully playable.

## Return Fields

Return these fields to `rote`, `rote-flow-crystallization`, or `rote-registry`:

- Flow name and path.
- Parameter contract and schema decisions.
- Test commands and representative inputs.
- Lint, release, index, search verification, and pending cleanup output.
- Registry target, visibility, and publish approval if sharing is requested.
- Published Play URI, execution readiness, blockers, published-reference execution-verification
  status and evidence, and visibility-based access guidance returned by `rote-registry` when the
  flow is published; otherwise none.
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
- Hands off to: live `rote guidance browser flow-authoring` for browser TypeScript;
  `rote-typescript-transformations` for non-browser TypeScript logic; `rote-registry` for sharing;
  `rote-flow-run` for final execution verification; `rote-troubleshooting` after repeated unchanged
  failures.
- Returns to: `rote` or `rote-flow-crystallization` with flow path, parameter contract, verification
  status, release state, verified Play URI and sharing guidance when published, and next owner.
- Stop when: the flow is verified, a release/publish approval is needed, a required schema or
  credential is missing, or troubleshooting becomes the correct owner.
- Completion signal: flow draft, release plus index/search verification and pending cleanup when
  applicable, publish handoff, or blocker is named with commands already run.
