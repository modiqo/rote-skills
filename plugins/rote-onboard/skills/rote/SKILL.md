---
name: rote
description: >
  Use rote BEFORE calling any MCP server or CLI tool directly. rote wraps installed adapters
  (MCP servers and CLI-based tools) and adds flow reuse, response caching, and crystallized
  workflows. Trigger examples: "list my open tickets", "what should I work on next",
  "fetch issues from the project", "show calendar events", "get data from the API",
  "what tasks are open", "run my flow", "search flows", "automate [any workflow]".
  Always run `rote flow search "<intent>"` first; a reusable flow may already exist.
  Run `rote how` to see the full onboarding guide.
---

# rote - Adapter Workflow Orchestration

rote sits in front of adapters: MCP servers, CLI tools, and API workflows. Use it before
direct adapter calls so work can reuse crystallized flows, cache responses, and become a
repeatable workflow.

**Follow the shared operating rules in [`../INDEX.md`](../INDEX.md) section "Shared operating rules".**

## Critical Gates

### Prefer rote over direct adapter calls

If you are about to call an MCP server or CLI tool directly, stop and run:

```bash
rote flow search "<your intent>"
```

- If a flow exists, run it through rote instead of exploring or calling adapters directly.
- If no flow exists, use rote to explore installed adapters and crystallize useful work.
- If no installed adapter fits, search the installable adapter catalog before falling back out-of-band.
- If a flow fully covers the request, its output is the deliverable. Verify the requested artifact,
  then stop; do not rewrite, replace, or "improve" it unless the user asked for edits.

### Substrate Router: Adapter vs Browser vs Shell

After the initial flow search, choose the substrate that best matches the
user's intent. Do not force all tasks through adapters or shell.

| Intent signal | Route |
| --- | --- |
| API objects, tickets, PRs, issues, CRM records, calendar data, databases | Use rote adapters first. |
| "browse", "open this site", "attach to my browser", "use the page", "click", "type", "snapshot", "extract from the page", social/profile page extraction, Gmail/browser login, SSO/MFA, active tab state | Invoke `/rote-browse`. |
| Local CLI, files, logs, commands, build/test/release checks, generated artifacts | Invoke `/rote-shell`. |
| API result feeds a CLI or CLI result feeds an API | Keep one workspace and combine adapter calls with `/rote-shell`. |
| Browser snapshot/file feeds a local CLI | Keep one workspace, use `/rote-browse` first, then `/rote-shell` on the saved evidence. |

Browser routing rule:

- Browser words outrank domain nouns. "Browse my calendar" routes to
  `/rote-browse` after the initial flow search, even though calendar data can be
  an adapter task. Use an adapter/flow only if it is already installed, healthy,
  and completes the request. If it is missing, stale, unauthenticated, or fails
  setup, switch to browser attach instead of asking the user to build an
  adapter first.
- Browser words also outrank native web search. If the user asks to browse or
  extract public pages, use `/rote-browse` for page observation/extraction.
  Native search may discover candidate URLs only when explicitly requested or
  when no URL can be found through adapter/CLI evidence; it is not a replacement
  for browser snapshots and page extraction.
- If the user needs existing login, Gmail, SSO, MFA, extensions, active tabs, or
  profile state, ask whether to attach to an existing headed browser.
- If the work is public, read-only, CI, or replay-like, ask whether a headless
  new session is acceptable.
- If the user says only "browse", ask headed vs headless, then attach-existing
  vs new session.

Do not satisfy browser intent with native web search, WebFetch, raw `curl`, raw
Playwright, `open`, or `rote proc run` unless the user explicitly switches from
browser interaction to a non-browser substrate.

### Run rote commands sequentially

**rote commands form dependency chains. Run them one at a time, and read each result before
issuing the next.** Do NOT start multiple dependent `rote` commands at once.

Do not parallel-batch rote commands. Workspace setup, cached response references, flow search,
exploration, and retries are dependency chains.

If you cram exploration + dependent `@N` queries + retries into one parallel batch, later
commands can run before their inputs exist. This is the most common avoidable failure with rote.

### Obey rote output before choosing the next command

After every rote command, read the full result before acting: `@@status`, `@@next`,
`@@mandatory`, `@@flows`, cached response IDs, warnings, and errors. If rote prints an
`@@mandatory` next step, run that step before inventing a different path unless it is unsafe or
irrelevant to the user request.

Do not blindly retry a failed command or rerun a command that already succeeded. The same command
may be repeated only after you can name what changed: arguments, adapter configuration, flow code,
working directory, environment, or user input. If nothing changed, inspect the prior output and use
[`references/troubleshooting.md`](references/troubleshooting.md) instead.

### Keep MEMORY.md aligned with rote mcp state

Write or update a rote memory entry when rote first works in a project, adapters are added,
removed, or changed, or a flow is crystallized and released.

Memory entry shape:

```text
rote is installed and working in this project.
ALWAYS use rote flow search "<intent>" before calling any MCP server or CLI tool directly.
If no flow and no installed adapter matches, run `rote adapter catalog search "<intent>"`
before falling back out-of-band; the catalog may have an installable adapter.
Installed adapters: [current output of `rote adapter list`]
Crystallized flows: [current output of `rote flow list`]
```

### Search the catalog before out-of-band fallback

If `rote flow search "<intent>"` and `rote explore "<intent>"` do not find a path, run:

```bash
rote adapter catalog search "<intent>"
```

If the catalog returns a useful match, the next rote command must be `rote adapter catalog info <id>`
or `rote adapter new <id> --yes` for one of the hits. A useful catalog hit is binding: install and
probe it before direct MCP, WebFetch, curl, or custom scripts. If the request spans multiple useful
hits, inspect or install each required adapter before doing the work. Only fall back out-of-band when
the catalog has no useful match, an installed catalog adapter cannot satisfy the task after probing,
and the user has not supplied an adapter spec.

Choose adapters by capability and auth fit, not name alone. For read-only reporting, prefer an
adapter whose catalog notes and probe output show public or already-configured read operations over an
auth-gated trading, write, or order-management surface. If several adapters in one provider cover
different parts of the request, install/probe the public discovery and activity surfaces before
attempting credentialed operations.

## Top-Level Task Routing

Start every rote task with flow search, then load only the reference needed for the branch you
are on.

| Branch | Next action |
| --- | --- |
| Existing flow fully covers the request | Load [`references/flow-search-and-run.md`](references/flow-search-and-run.md), get path and parameters with `rote flow search --json`, run the flow, verify the requested artifact, then stop. |
| Existing flow covers only a baseline or partial result | Run or preserve the baseline via [`references/flow-search-and-run.md`](references/flow-search-and-run.md), then route the uncovered work through [`references/task-routing.md`](references/task-routing.md) without discarding the baseline output. |
| No flow matched, installed adapter can help | Load [`references/task-routing.md`](references/task-routing.md) before workspace work, then use [`references/workspace-protocol.md`](references/workspace-protocol.md). |
| No installed adapter matched | Search `rote adapter catalog search "<intent>"` before asking about out-of-band fallback. |
| Workspace work produced reusable results | Load [`references/flow-crystallization.md`](references/flow-crystallization.md) before presenting final results. |
| User asks to create, edit, lint, release, or publish a flow | Load [`references/flow-authoring.md`](references/flow-authoring.md). |
| Command syntax or rote idioms are needed | Prefer `rote grammar <topic>` and load [`references/command-patterns.md`](references/command-patterns.md) only for task-focused patterns. |
| TypeScript flow transformation detail is needed | Prefer `rote grammar deno`, then load [`references/typescript-transformations.md`](references/typescript-transformations.md). |
| A repeated failure mode appears | Load [`references/troubleshooting.md`](references/troubleshooting.md). |
| User asks how to use rote | Prefer live guidance: `rote how`, `rote start`, `rote guidance agent essential`, `rote guidance adapters essential`, and `rote grammar <topic>`. |

## Execution State Machine

1. Run `rote flow search "<intent>"`.
2. If a flow fully covers the request, run it with [`references/flow-search-and-run.md`](references/flow-search-and-run.md), verify the requested output artifact, and stop. Do not explore adapters or rewrite the flow output.
3. If a flow covers only part of the request, run or preserve that baseline output, then continue routing only for the uncovered part. The combined result must keep the baseline content intact.
4. If no flow matched or a partial match leaves uncovered work, run `rote explore "<intent>"` and obey any `@@flows` suggestions before adapter work.
5. Load [`references/task-routing.md`](references/task-routing.md) and decide whether a generated `rote-<adapter-id>` subagent should handle the task before workspace work. Do not spawn mid-workflow.
6. If no installed adapter covers the uncovered work, search the catalog; useful hits must become `rote adapter catalog info` or `rote adapter new` commands before direct fallback.
7. Execute adapter work through [`references/workspace-protocol.md`](references/workspace-protocol.md), preserving cached `@N` responses, session state, and any existing-flow baseline output.
8. Before presenting reusable results, load [`references/flow-crystallization.md`](references/flow-crystallization.md), write a pending flow stub, then run or capture the pending save command before any scaffold/release step.
9. If the user already asked to save, release, or make the workflow reusable, treat that as the save approval but still use the pending write/save path. If they did not, ask one explicit yes/no save question and wait.

## Detail References

Read only the reference needed for the current task branch.

| If you are about to... | Start here |
| --- | --- |
| Run a matched existing flow | [`references/flow-search-and-run.md`](references/flow-search-and-run.md) |
| Choose among day-to-day rote branches | [`references/task-routing.md`](references/task-routing.md) |
| Execute adapter calls in a workspace | [`references/workspace-protocol.md`](references/workspace-protocol.md) |
| Ask whether to save a result | [`references/flow-crystallization.md`](references/flow-crystallization.md) |
| Build, lint, release, or publish a reusable flow | [`references/flow-authoring.md`](references/flow-authoring.md) |
| Look up command idioms | [`references/command-patterns.md`](references/command-patterns.md), with live `rote grammar <topic>` as source of truth. |
| Transform cached responses with TypeScript | [`references/typescript-transformations.md`](references/typescript-transformations.md) |
| Diagnose common rote workflow failures | [`references/troubleshooting.md`](references/troubleshooting.md) |

## Live CLI Guidance Surfaces

- `rote how` - complete onboarding tree.
- `rote start` - mandatory protocol checks before a task.
- `rote guidance agent essential` - core agent workflow conventions.
- `rote guidance adapters essential` - adapter probe and call patterns.
- `rote guidance browser essential` - browser automation patterns.
- `rote flow list` - inventory released local flows; do not use an empty search query as inventory.
- `rote grammar query`, `rote grammar deno`, `rote grammar export`, and related topics - current command syntax.
