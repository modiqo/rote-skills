---
name: rote-flow-authoring
description: >
  Create, edit, test, lint, release, verify, or publish reusable rote plays. Use for reusable
  contract elicitation, rote-driven schema discovery, scaffold/test/release/search lifecycle, and
  registry handoff.
---

# rote-flow-authoring

All `rote-<name>` references in this document — including every name in the Handoff
Contract — are companion **skills**, never CLI commands (`rote-shell` is not `rote shell`).
Invoke them through the runtime's skill mechanism; only literal `rote …` commands run in a
terminal.

Use this skill when the user asks to create, edit, lint, release, verify, or publish a reusable rote
play, or when `rote-flow-crystallization` returns an accepted or pre-approved plan plus its exact
pending scaffold command. A prepared pending-save command alone is not approval. Keep live command
help authoritative for syntax.

## Route The Implementation

Choose the implementation guide before editing the scaffold:

| Workflow concern | Required guidance |
| --- | --- |
| Browser navigation, snapshots, clicks, typing, browser auth, or replay | `rote guidance browser play-authoring` |
| Cached-response transformation or general TypeScript logic | `rote-typescript-transformations` plus `rote grammar deno` |
| Shell/process execution | The shell authoring route from `rote-shell` |
| Registry publication | `rote-registry` plus `rote grammar registry` |

Browser TypeScript does not belong to the generic transformation route. Run `rote guidance browser
play-authoring`, follow its typed-step, presentation, and legacy-SDK procedure, then return here for tests, lint, release,
index/search verification, and pending cleanup.

## Consume The Crystallization Plan

When this skill follows `rote-flow-crystallization`, treat its plan as the authoring input. The plan
must satisfy the canonical eight-field contract in `rote guidance play crystallization`, and the
decision must be accepted or pre-approved before materialization. Do not reconstruct the play from
transcript order, ask the save question again, or execute a prepared command whose decision is still
unclear. If the handoff lacks a usable plan or clear approval, return to crystallization; direct
authoring may read that guide before editing.

Use `rote guidance play shape` while mapping the plan into steps and
`rote guidance play testing` before release. Record any necessary deviation from the approved plan
in the final authoring return instead of silently changing the workflow contract.

## Elicit The Reusable Contract

Before scaffolding, identify the repeatable workflow boundary:

- User-visible goal and success condition.
- Required inputs, optional inputs, and safe defaults.
- Whether the play is atomic or a composed superplay that reuses an existing partial-play baseline.
- Adapter operations, browser actions, or shell/process operations needed to produce the result.
- Which captured response, file, or snapshot will back each required source and capability.
- Output shape a future caller should receive.
- Values that must remain parameters rather than hard-coded secrets, dates, IDs, or names.

For a composed superplay, keep the baseline play as a first-class reusable dependency in the
contract. Record the baseline play name, parameters, output artifact or structured output, and the
merge rule for uncovered work. Do not hard-code the baseline's last report contents into the new
play body unless the user explicitly asked for a static snapshot. The release test must prove the
output contains baseline evidence plus each newly required capability.

If the requirement depends on an API shape, discover it through rote adapter probes and calls in a
workspace before writing the play.

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
- For a schema-1 steps play, declare every single-segment `adapter/<id>` used by `steps[].endpoint`
  in `metadata.requires_endpoints` or the matching `metadata.mcp_servers` key. Fix
  `FLOW_ENDPOINT_UNDECLARED` before release. Schema-2 plays use their `adapter:` declaration. See
  `rote guidance typescript play-creation` for declaration forms.
- Test different server-side dimensions, not only the same pagination or output knob with different
  values.

## Choose The Play Shape

Reusable plays use steps + presentation by default. The effect plane is the frontmatter `steps:` DAG; the
presentation plane is the deprivileged TypeScript body selected by
`metadata.execution_model: steps_with_presentation`. Adapter, `process.exec`, and `browser.*`
effects all belong in `steps:`. The body only normalizes and renders the completed observations.

Use `rote grammar steps` for the step-syntax quick reference, the worked example in
`rote guidance typescript play-creation` for the complete assembled artifact, and live
`rote grammar export` plus `rote play template create --help` for exact scaffold syntax. Choose an
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
| Default steps + presentation play | The reusable workflow has adapter, process, browser, or mixed effects and needs stable human/summary/JSON output. | Frontmatter `steps:` AND `metadata.execution_model: steps_with_presentation`; the TypeScript body is presentation-only. | Use template/export with no shape flags, or run the command emitted by pending save unchanged. |
| Explicit steps-only DAG | The typed effect plan and runner report are the entire contract; no custom TypeScript rendering is wanted. | Frontmatter `steps:` WITHOUT `metadata.execution_model: steps_with_presentation`. | Request `--with-steps` for a template or `--format steps` for export, without a presentation flag. |
| Explicit legacy TypeScript, no `steps:` | The workflow needs control flow or runtime interaction the step language does not support (name it — see the representability test above), or the user explicitly requested a plain no-steps body. | No frontmatter `steps:`; effects live in the TypeScript body. | Request `--legacy-body` where live help advertises it — `template create` still requires `--adapter`, so an adapterless shell-only play has no generator: hand-author it from the legacy example in the rote-shell skill or `rote guidance typescript play-creation`; run with `rote deno run --allow-all`. |

These shape defaults do not migrate the adapter contract scheme. Schema v1 remains the default;
schema v2 is an explicit `--scheme 2` opt-in and must satisfy its own live contract/origin rules.

Do not put adapter calls, `Rote`, `runPreflight`, `fetch`, `Deno.Command`, direct env reads, or
filesystem access in a `steps_with_presentation` body. That body reads completed step outputs
through `__ROTE_PRESENTATION_SDK__`.

Before choosing package members or resource addressing, run
`rote guidance typescript play-creation` and apply its portable play-package contract. That live
guidance is authoritative for package roles, allowed resource positions, publication, identity,
and sensitive-data handling.

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

Use sibling steps for different independent effects. Use `for_each` when one operation is applied
over a set; on a local `for_each` its `max_concurrency` limits that fan-out and defaults to 10. Every
local command in one layer — plain siblings and fan-out children alike — also shares a layer ceiling
that defaults to 16, so a declaration is an upper bound and contention can admit fewer.
`--max-concurrency` does two things at once: it raises that layer ceiling when it asks for more than
the default, and it always caps the whole run across substrates. That second role is why a value below
the default never widens a layer. Plain concurrency is not a legacy-body
trigger. Browser work serializes its layer. See `rote grammar steps` for runtime semantics and
`rote guidance typescript play-creation` for examples.

`$param` substitutes into every step string field — adapter `params:` values and `process.exec`
`argv` elements alike; an unresolved token passes through as a literal, not an error. `@step{.path}`
reads a completed step's unwrapped body (never `.result.content[0].text`); `for_each: '$.items'`
fans a step out over the source step's array with `$item`/`$item_index` bound per element. Fan-out
width is resolved at run time, so a set discovered by an upstream step is NOT a legacy trigger —
over a process step's JSON stdout use `for_each: '$.stdout.text | fromjson'` (bridge example in
`rote guidance typescript play-creation`). A `process.exec` step takes an optional `timeout_ms:`
budget (default 30s). In the presentation body, play parameters arrive on `ctx.params` — never
echo them through a step's stdout and never read `Deno.args`.

### Preserve Process Failures

For `process.exec`, the child exit status is the DAG's failure signal; stdout is only data. If an
inline or invoked script catches a fatal error, write an actionable diagnostic to stderr and exit
nonzero or rethrow. Printing `{"ok": false}` to stdout and returning normally records a successful
step, so never use an error-shaped payload as a substitute for a failing exit status.

Exit zero for an expected optional degradation only when the output makes that distinction
explicit, for example `{"ok": true, "available": false, "warning": "..."}`. Classify every broad
exception handler as fatal or optional instead of converting all exceptions into successful
process results.

## Scaffold Through Rote

Use the pending save command from `rote-flow-crystallization` when authoring follows a completed
task. Crystallization must already have run pending write/save where supported and returned a decision
of accepted or pre-approved. Authoring executes the emitted scaffold command itself, immediately after
that handoff. A declined decision has already discarded the pending stub; an unclear decision belongs
back in crystallization. Never interpret approval independently in this skill.

The emitted scaffold command already encodes the pending artifact's steps + presentation shape. Run it
unchanged, exactly once. Never append shape flags, rebuild it from memory, or replace it with a second
scaffold command; doing so can discard the recorded adapters, browser sessions, workspace, or
execution model. Crystallization and substrate skills preserve this command but never execute it.

For adapter-backed or mixed adapter/shell work, "save this" means preserve workspace history, run
pending write/save, and scaffold from rote with `rote play template create`. It does not authorize
creating `~/.rote/flows/<name>` by hand or transcribing the last successful script into a play.

For direct authoring, use the current scaffold/export syntax from `rote grammar export`.
`rote play template create` is flag-driven: pass `--name <play-name>`, repeated
`--adapter <adapter-id-or-endpoint>` flags, and `--workspace <workspace-name>` when scaffolding from
workspace history. Do not create play directories by hand unless live rote guidance says to.

Preserve adapter ids from the pending stub and the routing decision. Do not swap to an adjacent
provider adapter during scaffold or implementation because the new id appears cleaner, newer, or
easier to call. If the selected adapter cannot satisfy the capability, stop authoring and return to
`rote-task-routing` with the missing capability.

For process-only work handed off by `rote-flow-crystallization`, do not invent an adapter just to
satisfy template or pending commands: those adapterless paths remain unsupported. Materialize the
approved plan through no-shape-flag workspace export over the recorded `rote proc` trace; it emits
`process.exec` steps plus presentation. If the
representability test above lands on a legacy body — a named gap in the step language, or an
explicit user request — author
`~/.rote/flows/<name>/main.ts` with `@rote-frontmatter`, create `deps.toml`, use the shell SDK, run
dependency preflight, and test that legacy body with `rote deno run --allow-all`.

A workspace export is a draft synthesized from one recording, not a finished play. `rote proc run`
records literal argv, so the export carries recorded literals (paths, owners, dates) and may lift
recorded values into spurious `parameters:` entries. Before lint: replace recorded literals with
`$param` tokens, declare each real parameter, and prune every inferred parameter that is not part
of the contract. Generalizing parameters and the presentation body is expected; changing the shape
by hand (adding/removing `steps:` or `execution_model`) is not — re-export with the explicit flag
instead. The "run the emitted command unchanged" rule protects the pending-save *scaffold command*,
not the exported artifact.

## Play Runtime Boundary

Play code already runs under its owning runner (`rote play run` for any `steps:` play, bundled Deno
only for an explicit no-steps legacy body); do not recreate rote lifecycle inside it.

- Do not run `rote init` from inside a TypeScript play.
- Do not shell out to `rote init` as a workaround for `Rote.workspace(...)` failures.
- Do not add `rote proc run` calls to a reusable play body.
- Workspace bootstrap belongs outside play execution unless the SDK owns it.

If `Rote.workspace(...)` creates a nonexistent cwd or fails, treat it as SDK/runtime failure.
Inspect `rote deno status`, `rote sdk status`, and `rote guidance typescript play-creation`.

## Implement The Play Body

For browser TypeScript, run `rote guidance browser play-authoring`. For non-browser TypeScript logic,
hand off to `rote-typescript-transformations` or follow its rules before continuing this lifecycle.

For plays whose frontmatter declares `metadata.execution_model: steps_with_presentation`, the
TypeScript body is presentation-only. Import `__ROTE_PRESENTATION_SDK__`, read completed outputs
with `loadPresentationContext()` and literal `stepName("...")` references, and render with
`FlowOutput`. Read the play's invocation parameters from `ctx.params`
(`Readonly<Record<string, unknown>>`, declared defaults already applied) — that is the sanctioned
input surface. Do not import the broad SDK, construct `Rote`, run preflight, create a task queue,
call `fetch`, spawn subprocesses, or read `Deno.args`/`Deno.env` directly from that body.

Live progress belongs to the runner because the presentation body starts only after declared steps
finish. A request for a spinner or staged progress must not move adapter calls into TypeScript,
remove `steps:`, or switch the play to a legacy body. Preserve the DAG and use a runner-owned
progress surface when available; otherwise state that live presentation progress is not supported
for this execution model instead of publishing a stepless substitute.

Normalize each observation without assuming a provider schema. Preserve `single`, `fan_out`, and
`empty_fan_out`; use `requireAvailable` for completed or checkpoint-restored output and branch on the
outcome for restored, failed, skipped, or blocked steps — a checkpoint-restored step arrives as its
own `restored` status, so a switch missing that arm breaks under `--resume`. Narrow process and browser bodies with
`isProcessExecBody` and `isBrowserBody`. Treat remaining adapter bodies as direct JSON, plain text,
or narrowly recognized legacy text-envelope residue; malformed/optional values stay explicit and
must not become fabricated empty strings, zeroes, false values, or arrays. Use the concrete pattern
in `rote guidance typescript play-creation`. If strict normalization requires representative input
during lint, use `rote guidance play testing` for fixture policy, then use `rote grammar steps` for
the exact frontmatter and manifest syntax. Fixtures are lint evidence, not executable step
configuration.

Preserve a stable `FlowOutput` shape:

- Return structured data for machine reuse.
- Include a concise human-readable summary when useful.
- For superplays, include enough structure to distinguish baseline-play output from newly added
  adapter/browser/process data.
- Keep adapter raw responses out of the final result unless the user needs them.
- Avoid embedding local workspace paths, secrets, or one-off IDs in output.

When the implementation needs local shell processing, use `rote proc` only during exploration.
Default steps + presentation plays put finite work in declarative `process.exec` steps; their presentation body
never runs shell SDK calls. Only an explicit no-steps legacy body uses `rote.exec`,
`rote.execBackground`, `rote.followFile`, and related wrappers. Never add raw child-process code,
shell pipelines, inline script snippets, or `rote proc run` to either body.

When the implementation needs provider/API data, default steps + presentation plays put adapter endpoint/method
calls in frontmatter `steps:` and presentation reads their typed observations. Only an explicit
legacy body uses adapter handles returned by `runPreflight`. Do not add raw HTTP fetches to bypass
missing adapter setup; fix the adapter route or hand back to `rote-task-routing`.

If an explicit legacy body's `rote deno run` fails because the generated SDK import cannot find
`mod.ts`, `Adapter.callBg` is undefined, or the runtime cannot find rote-managed Deno/SDK state,
treat that as a scaffold/runtime issue. Inspect `rote deno status`, `rote sdk status`, `rote grammar
deno`, and `rote guidance typescript play-creation`; do not hand-edit around it with raw Deno, raw
HTTP, or unrelated SDK APIs.

## Test, Lint, Release, And Search

Read `rote guidance play testing`, then run plays with frontmatter `steps:` (the default shape) through the play runner, from outside the
active workspace, with representative parameter sets. For presentation plays, it executes effects
first and then invokes the deprivileged presentation body:

```bash
rote play run /absolute/path/to/main.ts param=value
```

Run an explicit legacy TypeScript play, with no frontmatter `steps:` block, through bundled Deno
instead:

```bash
rote deno run --allow-all /absolute/path/to/main.ts [args]
```

The DAG runner takes named `key=value` parameters; the legacy body takes its declared positional
argument order.

Whichever runner applies, cover the common case, empty or no-result case, optional defaults, and
at least one user-provided edge case. For every process-backed play, also run a deliberately fatal
case and confirm that the command exits nonzero, the process step is `FAILED`, and the overall play
run fails. If that step has declared dependents, confirm they are `BLOCKED`; unrelated parallel
steps may still complete.

Release/lint success is not capability proof. Before release, rerun the capability check:
every required source is backed by a captured response, every adapter id from the routing decision
was installed and called, and the output contains baseline evidence plus each newly required
capability. Do not continue to release if lint/runtime succeeds while a required capability is
still missing.

Before calling the work complete, run the live lint/release path surfaced by rote. Release is a hard
gate, not a file edit. Execution success and `rote play validate` do not make a play discoverable;
only `rote play release <name>` performs the local lifecycle transition.
Never use `rote play release --force` to satisfy a save/release/publish task. A forced release is a
broken artifact state for humans to inspect, not an agent completion path.

Before release, obtain explicit authorization unless the original request already asked to release,
crystallize, finalize, make discoverable, save as reusable, or publish the play. After release,
verify discoverability:

```bash
rote play lint <name>
rote play release <name>
rote play index --rebuild
rote play search "<intent-or-play-name>"
rote play search "<intent-or-play-name>" --json
```

Before pending cleanup, verify the smoke run's final artifact content, including baseline markers
and newly required live/API sections.

If lint or release fails, change the play, arguments, adapter configuration, cwd, or environment
before retrying. Do not edit frontmatter `status` by hand.

If this authoring run came from a pending stub, cleanup is part of release completion:

```bash
rote play pending discard <workspace-name>
```

Run pending discard only after release succeeded, the index was rebuilt, and search verification
shows the released play. Do not leave `.pending-flow.json` or a workspace-scoped pending stub behind
as a resume anchor after the released play is discoverable.

## Publication Offer After Release

A successful release reports a `@@share` section (`data.share_nudge` under `--json`). It may carry a
one-time publication offer. It is an offer, not a gate: the release is already complete, and the offer
is made at most once for a released revision. Never read its presence as approval to publish, and
never infer approval to publish from the save decision, from the release, or from the offer appearing.

**Read `readiness` on every release, whatever else the section carries.** It describes the artifact
and is reported every time — including on a revision already declined or published, and on one whose
readiness could be assessed while publication was not on the table. It is never gated on an offer
being made:

- `blocked` — `readiness.blockers` names each blocker and the repair for it. Report every blocker and
  its repair, fix the artifact, and release again. A blocked artifact is never offered for
  publication, so it carries **no actions** — the blockers are the payload. Do not push it and do not
  describe publication as available.
- `not_assessed` — `readiness.gaps` names what could not be checked. Report what could not be
  checked. An unchecked artifact is not a ready one.
- `ready` — the artifact is fit to hand to someone else. That is all it means: `ready` is not an
  offer and says nothing about whether one was made.

**Whether publication was offered is a separate question, and only `actions` answers it.** On
`--json`, act only when `data.share_nudge.actions` is non-empty; the `Action:` lines are the human
rendering of that same collection. Those commands are deliberately absent from `@@next` and from
`data.next` — publishing and recording a decline each need the user's decision, so neither is a
"run this next" step, and nothing in `next` will ever offer you one. A `ready` readiness with no
actions means the offer for this revision is already spent or not available; say nothing about
publishing.

When actions are present, present the choice they name, then stop. Publishing needs the user's
explicit decision on target and visibility; `rote-registry` owns the push once they have made it.

A suggested push is always private, one per namespace you are authorized to publish to. Public
publication is always a separate, deliberate request; never widen visibility because the artifact
looked harmless.

If the user declines publication, record the decline so the offer is not repeated for this revision.
Do this **after the release completes** — the flow must already be released, or the command refuses:

```bash
rote play release <name> --keep-local
```

That records a decision and nothing else: it does not undo the release, does not touch the flow, and
does not block an explicit `rote registry play push` later. Run it only after an explicit decline. An
unanswered offer stays unanswered — silence is not a decline, and recording one the user did not make
suppresses the invitation they might have wanted.

Once a version is published the decline is refused, and the command still exits 0 — read the
`decision` field rather than the exit code: `published` means nothing was recorded.

## Registry Handoff

If the play should be shared, use `rote grammar registry` for the current push syntax. Confirm the
target namespace before publishing, then hand off to
`rote-registry` with play path, release status, owner/namespace, visibility, dependency notes, and
the user's publish approval. A local release alone has no published Play URI. When `rote-registry`
returns a `play_uri`, `bootstrap_uri`, resolved run reference, published-reference
`execution_verification` status and evidence, and access guidance (resolution and execution audiences) after
publication or an already-in-sync check, present and propagate them; do not construct or parse the
URI in this skill. Treat the disclosure-only Play URI, advertised bootstrap transition, static
execution readiness, and successful execution as separate facts: propagate `play_run_eligible`, the
execution variant, blockers, and verification status, and do not describe an unverified or failed
version as successfully playable.

## Return Fields

Return these fields to `rote`, `rote-flow-crystallization`, or `rote-registry`:

- Play name and path.
- Parameter contract and schema decisions.
- Test commands and representative inputs.
- Lint, release, index, search verification, and pending cleanup output.
- Registry target, visibility, and publish approval if sharing is requested.
- Publication offer: `offered` / `declined-and-recorded` / `suppressed` / `not offered`. Carry this
  even when nothing was published — `rote registry play push` does not consult the recorded
  decision, so a decline made here is invisible to `rote` and `rote-registry` unless you relay it,
  and they will raise sharing again.
- Published Play URI, execution readiness, blockers, published-reference execution-verification
  status and evidence, and access guidance (resolution and execution audiences) returned by
  `rote-registry` when the play is published; otherwise none.
- Next recommended skill: `rote-typescript-transformations`, `rote-registry`, `rote-troubleshooting`,
  or none.

## Handoff Contract

- Use when: a reusable play must be created, edited, tested, linted, released, verified, or prepared
  for publication.
- Preconditions: direct user intent defines the boundary, or `rote-flow-crystallization` supplies a
  usable plan, accepted/pre-approved decision, and exact pending scaffold command; any required API
  shape can be discovered through rote before scaffolding.
- Owns: contract elicitation, schema discovery, scaffold, implementation lifecycle, tests, lint,
  release, index/search verification, pending cleanup when applicable, and registry-ready return
  data.
- Hands off to: live `rote guidance browser play-authoring` for browser TypeScript;
  `rote-typescript-transformations` for non-browser TypeScript logic; `rote-registry` for sharing;
  `rote-flow-run` for final execution verification; `rote-troubleshooting` after repeated unchanged
  failures.
- Returns to: `rote` or `rote-flow-crystallization` with play path, parameter contract, verification
  status, release state, verified Play URI and sharing guidance when published, and next owner.
- Stop when: the play is verified, a release/publish approval is needed, a required schema or
  credential is missing, or troubleshooting becomes the correct owner.
- Completion signal: play draft, release plus index/search verification and pending cleanup when
  applicable, publish handoff, or blocker is named with commands already run.
