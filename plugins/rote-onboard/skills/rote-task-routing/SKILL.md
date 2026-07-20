---
name: rote-task-routing
description: >
  Choose the next rote execution path after flow search does not fully cover a request. Use for
  explore/catalog gates, shell/process routing, subagent-before-workspace decisions, single-adapter
  versus multi-adapter routing, and fallback boundaries.
---

# rote-task-routing

All `rote-<name>` references in this document — including every name in the Handoff
Contract — are companion **skills**, never CLI commands (`rote-shell` is not `rote shell`).
Invoke them through the runtime's skill mechanism; only literal `rote …` commands run in a
terminal.

Use this skill after `rote flow search "<intent>"` finds no runnable full match, or after
`rote-flow-run` returns a partial baseline that must be combined with additional work.

Do not use this skill after `rote-flow-run` verifies a full match. A verified full-flow result has no
uncovered work, so adapter exploration, catalog search, workspace setup, and fallback routing are
out of scope unless the user adds a new requirement.

This is a routing skill, not an execution or completion skill. It selects the next owner and returns
the handoff packet; lifecycle completion still belongs to `rote`, `rote-workspace`, `rote-shell`,
`rote-flow-crystallization`, or `rote-flow-authoring`.

Route once before workspace work begins, then keep the selected execution path stable. Do not begin
adapter calls and then hand off midstream unless the selected skill explicitly asks for that return.

## Capability Match Rule

Select routes by capability, not provider label. Before choosing an adapter, name the operation the
user needs: metadata/search, object details, file content, recent trades, live volume, state change,
browser session, or local process evidence.

- Before installing or calling anything, know for each required source the exact operation, the
  candidate adapter id, how the result will be verified, and the fallback boundary.
- If an installed adapter covers the provider but not the operation, keep searching.
- If an installed adapter covers the provider and operation but has a broken base URL, stale auth,
  or wrong config, repair it before replacing it.
- If a partial flow covers a baseline, route only the missing capability.
- If the catalog has several plausible provider adapters, run the Catalog Ambiguity Gate before
  installing or calling.
- Do not report provider metadata as live activity, event details as trades, or CLI scrollback as
  durable process evidence.

One provider commonly ships several adapters split by capability — for example entity/metadata
search in one, historical activity or trade/volume history in another, and live
orderbook/price/state in a third. Match the user's operation to the adapter that actually exposes
it, and repair an installed same-provider adapter (base URL, auth) before installing an adjacent
one.

## Catalog Ambiguity Gate

When catalog hits include multiple plausible same-provider adapters, or their auth shapes differ,
pause before install/call and ask the user to choose unless:

- the user named an exact adapter id;
- one candidate is already installed, healthy, and clearly covers every required capability;
- inspected metadata proves the others do not cover the needed operation.

Compare only decision fields: adapter id, kind/substrate, auth shape, spec-fetch auth, runtime auth,
capability fit, installed/health state, install/write impact, and next command.

The same gate applies when two or more **installed** adapters cover the required operation — e.g. a
REST adapter and an MCP adapter for the same provider. Do not silently take the first ranked hit:
ask the user to choose using the same decision fields, unless one candidate is unhealthy, the user
already picked one for this task, or inspected metadata proves only one exposes the needed
operation. Remember the choice for the rest of the task.

After `rote adapter catalog info`, hand off to `rote-adapter-create` for the install itself — it
owns the create pipeline. The routing-relevant rule: follow the catalog-provided install command;
if `Spec Type = "MCP"`, the route is `rote adapter new-from-mcp`. Do not substitute generic
`rote adapter new`.

A 401/403 while fetching an MCP/spec means spec-fetch auth is required; it is not proof the adapter
is bad. Keep it in the comparison instead of silently falling through to another adapter.

## Gate 1: Explore Before Adapter Work

Run:

```bash
rote explore "<intent>"
```

Read the full response before acting:

- If `@@flows` names a usable existing flow, hand off to `rote-flow-run`.
- If the uncovered work is local CLI, files, logs, build/test/release checks, generated artifacts,
  or process lifecycle, hand off to `rote-shell`.
- If the response identifies one installed adapter, choose the single-adapter branch.
- If the response requires multiple adapters or orchestration, keep the work in the main agent and
  use `rote-workspace`.
- If explore says no tools or no installed adapter fits, run an installed-adapter inventory before
  catalog install:

```bash
rote adapter list
rote adapter info <candidate-id>
```

Use this inventory to find provider-equivalent adapters that need repair. Search the catalog only
after no installed adapter can satisfy the capability or after repair is not possible.
- Do not route provider/API data collection to `rote-shell` just because a public HTTP endpoint can
  be called from a local script.

## Gate 2: Subagent Before Workspace

If an installed `rote-<adapter-id>` subagent exists and the task is scoped to that adapter, dispatch
it before any workspace calls start.

The handoff packet should include the user's goal, flow-search result, explore result, any baseline
artifact from `rote-flow-run`, and whether reusable output must pass through the save gate.

If the incoming artifact came from a full-flow match, return to `rote` instead of routing. Only
partial-flow baselines can be carried forward.

For a partial-flow baseline, preserve it as reusable source material for a composed superflow:
record the baseline flow name, parameters, output artifact, provenance, and the exact uncovered
requirements. The next execution owner should augment that baseline, not recreate a similar report
from scratch.

Do not spawn an adapter subagent after workspace calls have started. Mid-workflow handoff risks
losing cached `@N` responses and session context.

## Gate 2b: Shell/Process Work

Use `rote-shell` when the selected substrate is local commands, files, logs, generated artifacts,
dependency checks, or process lifecycle. Do not search the adapter catalog just because no adapter
covers a local CLI task; `rote proc` is the rote-native substrate for that work.

Build and dev-loop tooling is the exception, whatever the language toolchain: compilers, test
runners, formatters, linters, type checkers, package managers, and task runners run as plain shell
by default. Their value is the immediate pass/fail signal, and wrapping every iteration in
`rote proc` adds latency and noise without adding reusable evidence. If a specific build or test
run should become durable workspace evidence — it feeds a flow, a handoff, or a result the user
asked to keep — ask the user first with a one-line explanation: `rote proc` makes the output
queryable and replayable later. A preference the user or project instructions already state
settles the question without re-asking.

Provider/API access is not shell work. If the task needs provider records, event data, trades,
tickets, database rows, or other API objects, return to adapter routing and catalog search before
using `curl`, Python, Node, a provider SDK, or direct MCP tools. Shell routing may transform or
combine adapter/browser evidence after it has been captured through rote.

The handoff packet should include the working directory, input evidence, allowed local tools, any
baseline artifact that must be preserved, and whether the user already asked to save or release the
result. Process-only reusable work uses `rote-shell` and no-shape-flag workspace export for the
default steps + presentation artifact; template/pending commands still require a real adapter, so never invent
one.

Do not use shell routing as an excuse for raw, untracked parsing of rote responses. If the local
command consumes adapter/browser/flow evidence, route it through `rote-shell` and `rote proc` with
declared inputs/outputs so the transformation remains queryable and reusable.

## Gate 3: Single Adapter Without Subagent

If one installed adapter fits and no generated adapter skill exists, hand off to `rote-workspace`.
The workspace owns probes, calls, cached response queries, transformations, reusable-result
classification, and final workspace state.

Use `rote-using-adapters` when the runtime already selected a bundled single-adapter execution skill
for this exact branch and the handoff packet has enough state for delegated execution. Otherwise,
prefer `rote-workspace` so single-adapter and multi-adapter work share the same completion lifecycle.

## Gate 4: Multi-Adapter Orchestration

Keep multi-adapter work in the main agent. Use one workspace and sequence adapter probes, calls,
queries, and transformations there so cached responses remain addressable.

For multi-source requests, classify each source before work starts:

- Must use rote: the user explicitly says to use rote/adapters for that source, or the selected
  reusable workflow depends on that source.
- Direct one-off: only after flow search, explore, and catalog search do not produce a useful rote
  path, and the source is simple, public, unrelated to the reusable workflow, and not worth
  crystallizing.
- Needs catalog search: no installed adapter fits, but the source is part of the repeatable workflow.

Never bypass a source explicitly required through rote. Do not force unrelated one-off public data
through adapter installation unless it is part of the reusable workflow or the user asked for rote
coverage.

## Gate 5: Catalog Before Fallback

When no installed adapter or flow covers the task, run:

```bash
rote adapter catalog search "<intent>"
```

Use the user's domain words and operation words in the query. For a provider plus several data
needs, include both the provider and operations. If the first query returns partial matches, search
again with the missing capability before falling back.

If the catalog returns one unambiguous useful hit, inspect or install through rote with
`rote adapter catalog info <id>` or `rote adapter new <id> --yes`. If several hits plausibly fit,
run the Catalog Ambiguity Gate first. Treat the chosen hit as the next execution path: probe it and
call it through rote before direct MCP, WebFetch, curl, or custom scripts.

If you use `rote adapter new <id> ... --dry-run`, treat the result as a preflight only. The adapter
is not installed, its probe/call commands do not exist yet, and the task cannot be complete. Hand
off to `rote-adapter-create` for the real install (it follows the catalog-provided command,
including the MCP route), verify with `rote adapter info <id>` or `rote adapter list`, then run
`<id>_probe` and `<id>_call` for the required capability before answering.

Do not use catalog install to dodge a repairable installed adapter. A newly installed adjacent
adapter is a regression when the task expects the installed adapter's config to be fixed.

Only use direct tools after flow search, exploration, and catalog search fail to produce a rote path,
or after a catalog-installed adapter has been probed and cannot satisfy the request. Tell the user
what rote checked and why fallback is necessary.

## Return Fields

Return these fields to `rote` or the selected owner:

- Selected route: flow, adapter subagent, `rote-workspace`, `rote-using-adapters`, `rote-shell`,
  catalog install, or fallback.
- Explore result: relevant `@@status`, `@@next`, `@@flows`, installed adapter, or blocker.
- Catalog result: query used, hits inspected, installed adapter, or reason no hit fits.
- Baseline preservation: flow output/artifact that must remain intact.
- Requirements: required sources/adapters, missing capabilities, live observations, artifact,
  and verification checks.
- Workspace or shell preconditions: workspace name suggestion, working directory, adapter ids, local
  tools, credentials, and approvals.
- Next recommended skill: `rote-flow-run`, `rote-workspace`, `rote-shell`, `rote-using-adapters`,
  `rote-adapter-create`, or fallback to `rote`.

## Handoff Contract

- Use when: flow search did not fully cover the request, or a matched flow produced only a baseline
  for additional rote work.
- Preconditions: `rote` or `rote-flow-run` has provided the user intent, flow-search outcome, and any
  baseline output that must be preserved.
- Owns: explore-first routing, shell/process routing, subagent-before-workspace decisions,
  single-adapter versus multi-adapter selection, catalog search before fallback, and explicit route
  return fields. Does not own adapter execution, shell execution, flow authoring, or final
  presentation.
- Hands off to: `rote-flow-run` for `@@flows`; `rote-shell` for local CLI/files/logs/process work;
  `rote-workspace` for adapter workspace execution; `rote-using-adapters` for delegated
  single-adapter work; `rote-adapter-create` when catalog/spec work is needed; `rote-troubleshooting`
  when route selection repeatedly fails unchanged.
- Returns to: `rote` with the selected route, why it was selected, what rote checked, and which skill
  owns execution next.
- Stop when: a route owner is selected, no safe rote route exists, or the user must approve an
  adapter install, credential setup, or out-of-band fallback.
- Completion signal: route selected with handoff packet, or fallback/blocker explained with the rote
  checks already performed.
