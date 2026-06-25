# Task Routing

Use this reference after `rote flow search "<intent>"` finds no runnable flow, or after an existing
flow covers only the baseline and the prompt asks for additional data or automation. Route once
before workspace work begins, then keep the chosen execution path stable.

## 1. Explore before adapter work

Run:

```bash
rote explore "<intent>"
```

Read the full response before acting:

- If `@@flows` names a usable existing flow, switch to [flow-search-and-run.md](./flow-search-and-run.md).
- If the response identifies one installed adapter, choose the single-adapter branch below.
- If the response requires multiple adapters or orchestration, keep the work in the main agent.
- If no installed adapter fits, search the catalog before using out-of-band tools.

## 2. Single-adapter task with a generated adapter agent

If an installed `rote-<adapter-id>` subagent exists and the task is scoped to that adapter, delegate
before workspace work starts. Give the subagent the user's goal, relevant flow-search and explore
results, and the current save-gate requirement.

The subagent must follow the **`rote-using-adapters`** skill's single-adapter execution contract —
resume the already-selected route, then run its own workspace commands sequentially — and return
enough state for the main agent to continue the pending-stub save gate.

Do not spawn the subagent after workspace calls have started. Mid-workflow handoff risks losing
cached `@N` references and session context.

## 3. Single-adapter task without a generated agent

If no adapter-specific subagent exists, continue in the main agent with
[workspace-protocol.md](./workspace-protocol.md). Keep using rote commands rather than direct MCP or
CLI adapter calls.

## 4. Multi-adapter orchestration

Keep multi-adapter work in the main agent. Use one workspace and sequence adapter probes, calls,
queries, and transformations there so cached responses remain addressable.

For multi-source requests, classify each source before work starts:

- Must use rote: the user explicitly says to use rote/adapters for that source, or the already
  selected reusable workflow depends on that source.
- Direct one-off: the user did not require rote for that source and the source is simple,
  public, and not worth crystallizing as an adapter workflow.
- Needs catalog search: no installed adapter fits, but the source is part of the repeatable workflow.

Never bypass a source explicitly required through rote. Conversely, do not force unrelated one-off
public data through adapter installation unless it is part of the reusable workflow or the user asked
for rote coverage.

When the task needs multiple domains or multiple API surfaces in one domain, install/probe each
needed adapter through rote before considering direct fallback. A catalog hit for only one part of
the task is not enough to bypass rote for the rest.

If a later step becomes a clearly isolated single-adapter subtask, do not hand off unless no rote
workspace calls for the broader task have started. Prefer finishing the task and crystallizing the
repeatable workflow.

## 5. Catalog fallback

When no installed adapter or flow covers the task, run:

```bash
rote adapter catalog search "<intent>"
```

Use the user's domain words and operation words in the query, not just a broad category. For a task
that mentions a provider plus several data needs, include both the provider and the operations. If
the first query returns partial matches, search again with the missing capability before falling
back.

If the catalog returns a useful hit, first classify the install path:

- Setup path: the user asked to add, create, connect, or set up an adapter; the task needs a new
  credentialed adapter; auth is unclear or likely provider-specific; credentials are missing; or the
  catalog hit needs toolset/auth choices. Run `rote adapter catalog info <id>` and hand off to the
  **`rote-adapter-create`** skill. It owns dry-run-first analysis, auth research, and configured
  creation.
- Quick task-unblock path: the user is not asking for adapter setup, the adapter is just needed to
  finish the current task, and the catalog info or prior probe makes auth clearly public/no-auth or
  already configured. Only in this path, use `rote adapter new <id> --yes`.

Treat a useful hit as the next execution path: after installing, probe it and call it through rote
before direct MCP, WebFetch, curl, or custom scripts. Reading catalog info is not a substitute for
installing and using the adapter when it fits the task, unless classification routed the work to
`rote-adapter-create` first.

If multiple catalog hits map to the request, inspect or install all required adapters before work
starts. Use the catalog descriptions, authentication notes, and probe results to choose the right
API surface for each data need. Prefer public or already-configured adapters for read-only reporting;
do not choose an auth-gated adapter operation unless the environment has the required credentials or
the user explicitly accepts the auth setup.

When one provider has several adapters, separate read/reporting surfaces from write/trading/admin
surfaces. A name that sounds closer to the noun in the request is not enough if its useful operations
require missing credentials. For summaries, reports, digests, and audits, first try adapters whose
catalog notes and probes expose public list/search/activity/history operations; reserve credentialed
order-management or mutation surfaces for requests that actually need those actions.

Within one provider, different nouns may live on different public surfaces. For example, discovery,
market metadata, activity, trades, and history can be separate adapters. If the user asks for both
market selection and recent activity, inspect or install the public read surfaces for both needs
instead of assuming one adapter covers the whole provider.

Only use direct MCP, WebFetch, curl, or custom scripts after the catalog has no useful path, or after
a catalog-installed adapter has been probed and cannot satisfy the request.

## 6. Out-of-band escalation

Escalate out-of-band only after flow search, exploration, and catalog search fail to produce a rote
path. Tell the user what rote checked and why the fallback is necessary.
