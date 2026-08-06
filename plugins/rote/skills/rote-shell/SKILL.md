---
name: rote-shell
description: "Use for CLI and shell work through rote when command output, files, logs, process state, dependency checks, mixed API/browser evidence, or crystallized CLI plays must be remembered, queried, replayed, or shared."
---

# rote-shell

All `rote-<name>` references in this document — including every name in the Handoff
Contract — are companion **skills**, never CLI commands (`rote-shell` is not `rote shell`).
Invoke them through the runtime's skill mechanism; only literal `rote …` commands run in a
terminal.

Use rote for shell and CLI work when the result should become workspace memory:
commands, outputs, files, logs, process lifecycle, dependencies, and play replay.
Raw shell is still fine for tiny inspection commands, but work that matters
should pass through rote so it can be queried, audited, and crystallized.

Load the shell guidance when you need the complete command model:

```bash
rote guidance shell essential
```

## First Decision

Choose the narrowest rote primitive that preserves evidence:

- one-shot command: `rote proc run -- <program> [args...]`
- stdin from a file: `rote proc run --stdin-file input.txt -- <program>`
- declared output file: `rote proc run --capture-file label:path -- <program>`
- moving file/log: `rote proc stream follow --file logs/server.log --until READY`
- long-running process: `rote proc run --background --ready-log READY -- <program>`
- process stream: `rote proc stream follow-process proc-1 --stream stdout --until READY`
- terminal-sensitive command: `rote proc pty run -- <program> [args...]`
- dependency preflight: `rote deps check deps.toml`
- legacy TypeScript replay: `rote deno run --allow-all ~/.rote/flows/<name>/main.ts`
- declarative or presentation replay: `rote play run ~/.rote/flows/<name>/main.ts param=value`

Do not replace these with ad hoc `command > file`, `tail -f`, or `ps | grep`
when the evidence should be durable. Rote already stores typed responses,
artifacts, hashes, offsets, process leases, and command-log provenance.

Hard default: if command output feeds a final answer, user artifact, play
body, or later reasoning step, it is not disposable. Use `rote proc` and then
query the saved response.

Build and dev-loop tooling is the exception, whatever the language toolchain:
compilers, test runners, formatters, linters, type checkers, package managers,
and task runners run as plain shell by default. Wrap such a run in `rote proc`
only after the user agrees it should become durable workspace evidence.

Chaining is acceptable when each meaningful shell process is captured by `rote proc`.
For example, `rote proc run ... && rote proc run ...` preserves process nodes in the workspace DAG.
Do not promote raw pipes or chained lifecycle commands as the default route. Adapter setup,
workspace init/session setup, pending-play lifecycle, release, index, and registry operations are
clearer as separate commands unless the runtime explicitly prints a combined command.

## Adapter-First Boundary

`rote proc` is not an adapter substitute. If the task needs provider/API data,
such as records, tickets, PRs, issues, markets, trades, CRM rows, calendar
events, or database objects, return to `rote-task-routing` and use adapter
search/install/probe/call first. Do not fetch provider REST endpoints through
`curl`, Python, Node, a custom SDK, or inline HTTP scripts unless play search,
explore, catalog search, and adapter probes have failed to produce a rote path.

Shell work starts after routing selects local command evidence, or after
adapter/browser evidence has been captured and a local command needs to process
it. Provider-native CLIs are shell work only when the CLI itself is the requested
source or the router selected it as fallback after adapter checks.

## Handoff Contract

- Use when: local CLI commands, files, logs, process lifecycle, dependency checks, generated
  artifacts, or shell-derived play replay need rote memory.
- Preconditions: `rote` can run; a task intent and working directory are known; browser/API state
  that must feed the command has been materialized as a saved response, snapshot, slice, or file.
- Owns: `rote proc` primitive selection, process evidence capture, dependency preflight,
  background lease handling, process-backed TypeScript play shape, and shell SDK guidance.
- Hands off to: `rote-flow-crystallization`, `rote-flow-authoring`, `rote-typescript-transformations`,
  `rote-troubleshooting`, `rote-browse`, and `rote`.
- Returns to: `rote` or the delegating companion with commands run, response IDs, process leases,
  captured artifacts, result, reusability signal, cleanup state, and next recommended skill.
- Stop when: the shell result is delivered, dependency provisioning needs user approval, a process
  cleanup or credential prompt blocks, browser/API ownership is required, or reusable play authoring
  takes over.
- Completion signal: queried process evidence, captured files/artifacts, resolved background
  leases, dependency check status, and a save gate of pending, accepted, discarded, or not
  applicable.

## Handoff Packet

Consume this packet from `rote`, `rote-browse`, `rote-workspace`, or a delegated skill:

- Origin skill: `rote`, `rote-browse`, `rote-workspace`, or the delegated skill that handed off.
- User intent: the exact shell-visible result or artifact requested.
- Workspace path: current rote workspace path, or proposed workspace name.
- Working directory: directory where local commands should run.
- Input evidence: saved responses, browser snapshots/slices, files, or adapter results to consume.
- Requirements: the command output, files/logs, output artifact, and verification checks that
  must survive handoff.
- Allowed commands: exact local CLIs or `rote proc` primitives expected.
- Stop conditions: missing dependency, prompt/credential request, unsafe mutation, cleanup failure,
  or user approval needed.
- Return fields: commands run, response IDs, process leases, captured artifacts, cleanup state,
  result, reusability signal, save gate, and next recommended skill.

## Handoff Summary

Write or return this summary when shell work must survive handoff, compaction, or background waits:

```markdown
# Rote Shell Handoff Summary

- Active skill: `rote-shell`
- Origin skill: `rote` or `rote-...`
- User intent: ...
- Workspace path: ...
- Working directory: ...
- Commands run: ...
- Cached responses: ...
- Process leases: ...
- Captured artifacts: ...
- Requirements: ...
- Current gate: shell execution, dependency approval, background wait, cleanup, save gate, or blocker
- Result: ...
- Save gate: pending, accepted, discarded, or not applicable
- Next skill: `rote-flow-crystallization`, `rote-flow-authoring`, `rote-troubleshooting`, `rote`, or none
- Blockers: ...
- Completion signal: ...
```

## Browser Intent Router

If the user says "browse", "open this site", "attach to my browser", "use the
page", "click", "type", "snapshot", "extract from the page", "extract social
profiles", "Gmail in browser", or otherwise asks for live web UI state, stop
shell routing and invoke `/rote-browse`.

Do not satisfy browser intent with `rote proc run`, raw Playwright, native web
search, WebFetch, `open`, `curl`, or a saved non-browser play unless the user
explicitly switches substrate. Native search may help discover a URL only when
the user asked for search/discovery or the browser route cannot identify a URL;
it is not a substitute for browsing and extracting the page with rote.

Browse intent has precedence over the domain noun. For example, "browse my
calendar", "browse Gmail", "browse HubSpot", or "browse Salesforce" means the
user wants browser-state access even though those domains may also have APIs.
You may still run `rote play search "<intent>"` first, but only use an
adapter/play if it is installed, healthy, and completes the request. If the
adapter/play is missing, stale, unauthenticated, or fails setup, do not ask the
user to build an adapter before trying the browser route. Hand off to
`/rote-browse` and use an existing headed browser when the task depends on the
user's logged-in profile.

For public profile extraction, such as "browse each committer's social
profile", use GitHub/CLI/API data to collect candidate URLs and identities,
then use `/rote-browse` to visit and extract the public pages. Do not replace
that browse step with native web search summaries.

Default browser decision:

- Existing login, Gmail, SSO, MFA, extensions, active tabs, or profile state:
  ask to attach to an existing headed browser.
- Public read-only page, CI, or replay-like work: ask whether headless
  new-session is acceptable.
- The user says "browse" without choosing mode: ask headed vs headless; if
  headed, ask attach-existing vs new rote-managed headed browser.

For active browser attach, hand off to `/rote-browse` and use its sequence:

```bash
rote browser attach setup --method extension --browser chrome
rote browse <url> --headed --attach-existing --new-tab --no-prompt --no-snapshot
rote browse wait --selector '<ready-selector>' --timeout 30 --quiet-ms 750
rote browse snapshot
```

For browser plus shell work, keep both in the same workspace. Use
`/rote-browse` for page leases, snapshots, slices, refs, auth state, and
readiness. Use `/rote-shell` only after browser state has been materialized as
a saved response, snapshot, slice, or file that a local CLI should process.

## Current Capability Map

Use only shipped commands. Do not invent aliases from the roadmap.

| Need | Shipped command | Notes |
| --- | --- | --- |
| One-shot process capture | `rote proc run -- <program> [args...]` | Direct argv by default. |
| File stdin | `rote proc run --stdin-file input.txt -- <program>` | Records stdin provenance. |
| Declared output file | `rote proc run --capture-file label:path -- <program>` | Captures file metadata and artifact pointer. |
| Saved stdout/stderr files | `rote proc run --stdout-file out.txt --stderr-file err.txt -- <program>` | Use for durable stream files. |
| Dependency preflight | `rote deps check deps.toml` | No install side effects. |
| Moving file/log stream | `rote proc stream follow --file app.log --until READY` | Supports offsets, chunks, hashes, and pattern stop. |
| Background start | `rote proc run --background --ready-log READY -- <program>` | Creates a tracked lease such as `proc-1`. |
| Background status | `rote proc status proc-1` | Query lease state before acting. |
| Background wait | `rote proc wait proc-1 --timeout-ms 300000 --poll-ms 500` | Blocks until a tracked finite job exits or times out; records exit plus stdout/stderr observations. |
| Background stdout/stderr follow | `rote proc stream follow-process proc-1 --stream stdout --until READY` | Reads from background log artifacts. |
| Background stop and cleanup | `rote proc stop proc-1` | On Unix, stops the process group and records cleanup facts. |
| One-shot terminal transcript | `rote proc pty run -- <program> [args...]` | Use when the command must see a terminal. |

Deferred or not shipped as commands yet:

- `rote proc attach`
- `rote proc log`
- explicit `detach`
- persistent PTY sessions: start, send, snapshot, stop, attach
- non-log readiness probes such as HTTP, TCP, file, or command probes
- direct OS-pipe stream handles before background output reaches log artifacts
- non-Unix process-group cleanup guarantees

If a task needs a deferred feature, say so and use the nearest shipped primitive
instead. For example, use `rote proc wait` for finite tracked jobs, use
`rote proc stream follow-process ... --until <pattern>` for log observation or
readiness, and use `rote proc status` plus `rote proc stop` instead of raw
`ps`/`kill` when the process is tracked.

## TypeScript SDK Pattern Map

For explicit legacy no-steps TypeScript plays, use first-class SDK wrappers instead of
hand-assembling command arrays. Recorded finite commands default to typed `process.exec` steps via
workspace export; the SDK surface below is for the shell capabilities the step language does not
cover — PTY interaction, and background leases/streams with concurrent mid-lease work:

| Pattern | SDK call |
| --- | --- |
| Durable one-shot | `await rote.exec({ argv: ["git", "status", "--short"], deps: ["git"] })` |
| Declared stdin/output files | `await rote.exec({ argv, stdin: { file }, capture: { stdout: { file }, files: [{ label, path }] } })` |
| Dependency preflight | `await rote.depsCheck({ manifest: "deps.toml" })` |
| Tracked background job / detach-like work | `await rote.execBackground({ argv, readyLog, readyTimeoutMs, capture })` |
| Long-running job with useful work to do *while it runs* | `await rote.execBackgroundAndJoin(request, async (job) => { ... }, { timeoutMs, pollMs, stopOnWorkError })` |
| Lease status | `await rote.execStatus("proc-1")` |
| Lease wait | `await rote.execWait("proc-1", { timeoutMs: 300_000, pollMs: 500 })` or `await job.wait(...)` |
| Lease cleanup | `await rote.execStop("proc-1")` |
| Moving file stream | `await rote.followFile("logs/app.log", { until: "READY" })` |
| Background process stream | `await rote.followProcess("proc-1", "stdout", { until: "READY" })` |
| One-shot PTY transcript | `await rote.ptyRun({ argv, cols: 100, rows: 30 })` |
| Authored ordered fan-out | `await rote.execMany(requests, { stopOnError: false })` |

Use `rote.shell().<method>` when you want the shell namespace explicitly; the
top-level `rote.<method>` forms are convenience aliases for authored plays.

Do not invent SDK methods for deferred roadmap items. There is no
`rote.detach`, persistent PTY `send`, or direct OS-pipe stream handle yet. The
current detach-like pattern is a tracked background lease with stdout/stderr
files, `execWait`, `followProcess`, `execStatus`, and `execStop`.

Reading an exec result — every read is awaited; `stdout`/`stderr` are stream
handles, not strings:

```typescript
const proc = await rote.exec({ argv: ["git", "status", "--short"], deps: ["git"] });
const text = await proc.stdout.text();   // stderr mirror: proc.stderr.text()
const exit = await proc.exit();          // { kind: "code", code } | { kind: "signal", … }
if (exit.kind !== "code" || exit.code !== 0) {
  throw new Error(`git failed: ${await proc.stderr.text()}`);
}
const files = await proc.files();        // captured file artifacts ([] unless capture requested)
```

For declarative `process.exec`, the child exit status is likewise the DAG's control-plane result.
If an inline Python, shell, or JavaScript program catches a fatal error, write the diagnostic to
stderr and exit nonzero or rethrow. A stdout payload such as `{"ok": false}` followed by exit zero
is successful process data and cannot fail the step. Reserve exit-zero degradation for expected
optional absence, represented explicitly as success such as
`{"ok": true, "available": false, "warning": "..."}`.

`rote.execMany` preserves workspace response ordering by running process
requests serially. For several different independent commands, generate sibling
`process.exec` steps with no `depends_on` between them; for one command over many
values, generate declarative
frontmatter `steps:` with `type: process.exec`, `for_each`, and
`max_concurrency` so the DAG runner owns the scheduling and provenance.

For long-running finite jobs in authored TypeScript, prefer
`execBackgroundAndJoin` or `job.join` when there is useful adapter, browser,
process, file, or explicit stream work to do while the lease runs. The callback
creates normal semantic DAG evidence; the final `proc wait` is the join point.
Do not generate heartbeat or polling loops as DAG work.
`rote.execBackground(...)` already prints the lease and poll commands to
stderr. Do not duplicate that announcement in crystallized plays. Use
`announce: false` only for deliberately quiet plays.

## Crystallization Router

When the user asks to turn recorded shell exploration into a reusable play, use
`rote workspace export <path>` with no shape flags first. The default artifact is
schema-v1 steps + presentation: typed `process.exec` effects in `steps:` plus a
`steps_with_presentation` body. The export is a draft synthesized from one
recording: `rote proc run` records literal argv, so replace recorded literals
(paths, repos, dates) with `$param` tokens and prune any spurious inferred
`parameters:` entries before lint. Use explicit `--format steps` only when the
user wants a steps-only report. Adapterless template/frontmatter/pending
commands still require a real adapter, so do not invent one for process-only
work. Step syntax quick reference: `rote grammar steps`.

Use an explicit legacy no-steps shell body only when the workflow needs shell
control flow or runtime interaction the step language does not support, or when
the user asks for one. On this substrate that means PTY interaction and
background leases/streams with concurrent mid-lease work; "dynamic" on its own is
not a trigger. `rote-flow-authoring` owns the general representability test:

| Exploration pattern | Crystallized shape | Author via |
| --- | --- | --- |
| One finite command whose output is the fact | Default steps + presentation export with a typed `process.exec` step | No-shape-flag `rote workspace export ~/.rote/flows/<name>/main.ts` |
| Command writes files that downstream work reads | Default steps + presentation export with declared `process.exec` captures | No-shape-flag `rote workspace export ~/.rote/flows/<name>/main.ts` |
| A moving file or log must be followed to an until/readiness condition (the stream capability) | Explicit legacy body with `rote.followFile(path, options)` | Hand-author `main.ts` (legacy example below); run via `rote deno run --allow-all` |
| Long finite job where other useful work runs mid-lease (the concurrent-work capability) | Explicit legacy body with `rote.execBackgroundAndJoin(request, async (job) => { ... }, options)` | Hand-author `main.ts` (legacy example below); run via `rote deno run --allow-all` |
| Long service or daemon the play interacts with while it runs | Explicit legacy body with `rote.execBackground({ readyLog, capture })`, then status/follow/stop | Hand-author `main.ts` (legacy example below); run via `rote deno run --allow-all` |
| Need to inspect progress mid-lease, inside a justified background escape | Explicit legacy body with `job.follow(...)` or `rote.followProcess(...)` | Hand-author `main.ts` (legacy example below); run via `rote deno run --allow-all` |
| Need completion proof mid-lease, inside a justified background escape — a finite command with no concurrent work is a foreground `process.exec` step with `timeout_ms:` | Explicit legacy body with `job.wait(...)` or `rote.execWait(...)` | Hand-author `main.ts` (legacy example below); run via `rote deno run --allow-all` |
| Terminal behavior is the point | Explicit legacy body with `rote.ptyRun({ argv, input, cols, rows })` | Hand-author `main.ts` (legacy example below); run via `rote deno run --allow-all` |
| Several independent commands run at once | declarative `steps:` with sibling `process.exec` steps, no `depends_on` between them | Export first, then drop the synthesized `depends_on` edges per `rote grammar steps` |
| Many independent commands share one shape | declarative `steps:` with `process.exec`, `for_each`, and `max_concurrency` | Export first, then add `for_each`/`max_concurrency` per `rote grammar steps` |
| The item set is discovered at run time by another command or API call | declarative `steps:` fan-out sourced from the upstream step | NOT a legacy trigger; `rote grammar steps` has the bridge example for a process step's JSON stdout |
| One finite command needs more than the default 30s step budget | `timeout_ms:` on the `process.exec` step, or per-item fan-out so each item gets its own budget | `rote grammar steps` documents the budget |

Crystallize causality, not waiting. Do not encode heartbeat loops, repeated
status polling, or sleep/retry scaffolding as business DAG nodes. Those are
observation mechanics. The reusable play should expose semantic actions and
joins: start work, observe meaningful artifacts or streams, wait for completion,
then summarize.

Before authoring a process-backed TypeScript play, read:

```bash
rote guidance typescript play-creation
```

That guide owns frontmatter, `deps.toml`, FlowOutput, release QA, and the shell
SDK wrapper contract.

### Play Runtime Boundary

`rote proc run` is for exploration, not reusable play bodies.

- Process-backed TypeScript plays must not call `rote init` internally.
- Crystallized TypeScript plays must not shell out to `rote proc run`.
- During exploration, `rote proc run` records evidence; inside reusable plays, use SDK wrappers or
  `process.exec` steps.
- Use shell SDK exec only for task commands, not lifecycle bootstrapping.

## Strategy Pattern Library

Map the user's vague request to the smallest shipped pattern that preserves
evidence:

| Signal | Pattern | Use |
| --- | --- | --- |
| Tiny disposable inspection | Raw shell allowed | direct harness shell |
| Result may be queried, compared, summarized, or replayed | Durable one-shot | `rote proc run --` |
| Command reads a known file | Declared file input | `rote proc run --stdin-file` or a direct argv path |
| Command creates a file that matters | Declared file output | `rote proc run --capture-file` |
| Full stdout/stderr matters | Durable stream files | `--stdout-file`, `--stderr-file` |
| Required tools or input files matter | Dependency gate | `deps.toml` plus `rote deps check` |
| Existing log is moving | File stream watch | `rote proc stream follow --file` |
| Start a server or daemon-like process | Service lease | `rote proc run --background --ready-log` |
| Long finite non-interactive job | Tracked background job | `rote proc run --background --stdout-file --stderr-file` |
| Inspect background output | Process stream observation | `rote proc stream follow-process` |
| Need liveness before acting | Lease status | `rote proc status` |
| Need cleanup | Lease cleanup | `rote proc stop` |
| Many independent items share a command shape | Fan-out batch | `steps:` with `process.exec`, `for_each`, and `max_concurrency` |
| Item set discovered at run time by an upstream step | Fan-out over step output | `steps:` with `for_each` sourced from the upstream step — not a legacy trigger; see `rote grammar steps` |
| API result feeds CLI or CLI result feeds API | Mixed substrate chain | adapter/browser primitive plus `rote proc run` |
| Browser snapshot/file feeds local CLI | Browser-file bridge | browser snapshot/file plus `rote proc run` |
| Release, publish, deploy, or global mutation | Guarded mutation | deps check, tool-native dry-run, approval, then foreground unless background is approved |
| Command checks whether stdout is a terminal | One-shot PTY transcript | `rote proc pty run --` |
| Command may prompt interactively and can be scripted safely | One-shot PTY with bounded input | `rote proc pty run --input` or `--stdin-file` |
| Command needs ongoing human interaction | Foreground or defer | persistent PTY attach is not shipped |

`--dry-run`, `--resume`, `--force-resume`, and `--max-concurrency` are DAG
runner controls for frontmatter `steps:` plays. They are not universal
`rote proc run` flags. For a single command, use the tool's own dry-run/check mode
when available.

## Workspace Setup

CLI work must happen inside a rote workspace. Command runners rarely preserve cwd between steps,
so prefix each workspace-scoped command with the resolved `cd <workspace-path> && ` (one logical
step):

```bash
rote init cli-work --seq --force
cd ${ROTE_HOME:-$HOME/.rote}/rote/workspaces/cli-work && rote model set <model> --provider <provider> --confirmed-current
cd ${ROTE_HOME:-$HOME/.rote}/rote/workspaces/cli-work && rote workspace sandbox cli-work off
```

If the runtime does not expose the current model and provider and no command requires identity,
record that they were unknown; do not fabricate model metadata.

Keep commands simple and literal. Avoid shell control operators, command
substitution, and long `&&` chains unless the user explicitly needs shell
semantics (the resolved `cd <workspace-path> && rote …` prefix is the standing
exception). Prefer direct argv:

```bash
rote proc run -- rg TODO docs
```

Use shell parsing only when it is the real subject of the work:

```bash
rote proc run -- sh -c 'printf "alpha\n" | tr a-z A-Z'
```

If parsing transforms evidence for a report or play, keep it inside a
tracked command and capture the inputs/outputs. Do not rely on terminal
scrollback, untracked temp files, or raw pipelines as the only record.

## Query After Every Meaningful Step

Treat every `@N` as the handle for what just happened. The handle is saved
evidence, not always the raw business payload. Route by type.

After a command finishes, read `@@result` first:

- `response_id` is the saved evidence handle
- `response_kind` tells you how to query the evidence
- `primary_query`, `primary_stdout_query`, `primary_stderr_query`, or
  `artifact_query` are exact typed queries for the saved response
- `@@next` remains the immediate next-action guide

For one-shot process output, prefer the typed query fields from `@@result`:

```bash
rote query @1 '.stdout.text' -r
rote query @1 '.stderr.text' -r
rote query @1 '.status.exit' -r
rote query @1 '.files' -r
rote query @2 '.cleanup' -r
```

Use `@proc` addresses when inspecting process responses:

```bash
rote query @proc.last.stdout '.text' -r
rote query @proc.1.exit '.' -r
rote query @proc.last.transcript '.text' -r
```

When `@@result` gives an exact query such as
`rote query @proc.4.transcript '.text' -r`, prefer it over `@proc.last` if any
other command may run before you query. Use `rote @N .` for full provenance
inspection, not as a shortcut for stdout or PTY transcript text.

Do not summarize from terminal scrollback when a structured query exists.

## Files And Artifacts

Declare file inputs and outputs instead of relying on memory:

```bash
rote proc run \
  --stdin-file input.txt \
  --capture-file summary:out/summary.txt \
  -- python3 scripts/summarize.py
```

This records stdin provenance, file change state, media type, hashes when
available, and artifact paths under `.rote/artifacts/processes/@N/`.

## Background Processes

Start servers as leases, not mystery PIDs:

```bash
rote proc run --background --ready-log "Listening" --ready-timeout-ms 10000 -- npm run dev
rote proc status proc-1
rote proc stream follow-process proc-1 --stream stdout --until "GET /health"
rote proc stop proc-1
rote query @3 '.cleanup' -r
```

On Unix, rote starts background commands in a process group and stops the group
with `TERM` followed by `KILL` if needed. Inspect `.cleanup` before claiming a
process stopped.

## Long-Running Finite Jobs

For long-running but finite commands, use a tracked background lease instead of
an untracked detach. `rote proc detach` is not shipped yet. The current
detached-like primitive is:

```bash
rote proc run \
  --background \
  --stdout-file logs/job.stdout.log \
  --stderr-file logs/job.stderr.log \
  -- cargo test --all-targets --all-features

rote proc status proc-1
rote proc wait proc-1 --timeout-ms 600000 --poll-ms 1000
rote proc stream follow-process proc-1 --stream stdout --from-start --max-bytes 65536
rote proc stream follow-process proc-1 --stream stderr --from-start --max-bytes 65536
```

Use this when the command is non-interactive, can safely continue while the
agent does other work, and its stdout/stderr are enough to monitor progress.
Use `proc wait` to establish completion and exit status. Use
`follow-process` to inspect output, not as the primary completion detector.
Keep the lease id (`proc-1`) in the task notes and query status before making
claims about completion if wait has not finished.

In authored TypeScript, prefer the semantic join helper:

```ts
const joined = await rote.execBackgroundAndJoin(
  { argv: ["cargo", "test"], deps: ["cargo"] },
  async (job) => {
    const checks = await rote.exec({ argv: ["gh", "pr", "checks"], deps: ["gh"] });
    const stderr = await job.follow("stderr", { fromStart: true, maxBytes: 65536 });
    return { checks, stderr };
  },
  { timeoutMs: 600_000, pollMs: 1_000, stopOnWorkError: true },
);
```

The callback's work becomes semantic DAG evidence. The wait remains the
completion join. Poll/heartbeat internals do not become graph nodes.

Do not background a command when it may prompt for credentials, OTP, passphrase,
license acceptance, or confirmation. `rote proc pty run` can capture a bounded
one-shot terminal transcript, but it does not provide persistent attach. Ongoing
interactive jobs should run foreground or be postponed until the user can drive
the prompt.

## One-Shot PTY

Use PTY only when terminal behavior is the point: commands that check
`isatty`, render progress differently on terminals, or can be driven with
bounded scripted input.

```bash
rote proc pty run --cols 100 --rows 30 -- script-that-checks-tty
rote proc pty run --input "yes\n" -- interactive-but-scriptable-command
rote query @proc.last.transcript '.text' -r
```

Do not pass secrets through `--input`; terminal echo may record them in the
transcript. Do not use PTY for persistent REPLs or long human-driven sessions
until start/send/snapshot/stop support exists.

In a legacy play body, `rote.ptyRun` returns the same evidence, typed:

```typescript
const proc = await rote.ptyRun({ argv: ["sh", "-c", "test -t 1 && echo interactive || echo piped"], cols: 100, rows: 30 });
const transcript = await proc.transcript.text(); // raw transcript, ANSI escapes included
const exit = await proc.exit();                  // { kind: "code", code } | { kind: "signal", … }
if (exit.kind !== "code" || exit.code !== 0) throw new Error("capture failed");
```

Request fields are `argv`, `input` or `stdinFile` (not both), `cols`, `rows`,
and `timeoutMs`. There is **no `cwd` option** — the command runs in the
executor's current directory, which is the workspace directory once
`Rote.workspace()` is open. Pass directories to the tool itself (`git -C
"$root"`, `sh -c 'cd "$1" && …' sh "$dir"`) instead of relying on the play's
launch directory. Lint mode stubs `proc.exit()` (`{ kind: "code", code: 0 }`)
and `proc.transcript.text()` (`""`), so PTY reads need no `isLintMode()` guard.

Treat release and publish commands as high risk. For examples like
`cargo release`, `npm publish`, `gh release create`, or deploy commands:

1. Run dependency preflight first.
2. Run the tool's dry-run/check mode first when available.
3. Show the planned mutation and ask the user for approval before the real run.
4. Prefer foreground for final irreversible publish steps unless the user
   explicitly approves tracked background execution.
5. If background execution is approved, capture stdout/stderr to files, follow
   process streams, and keep the play `draft` until completion evidence is
   captured.
6. Do not stop a release/publish process unless the user asks; interruption can
   leave external state partially mutated.

Decision rule:

- short and non-interactive: foreground `rote proc run`
- long and non-interactive: tracked `rote proc run --background`
- long-running service: tracked `rote proc run --background --ready-log ...`
- interactive: foreground or wait for PTY/attach support
- irreversible release/publish: dry-run, user approval, then foreground unless
  tracked background is explicitly requested

## Dependency Manifests

When a task depends on local tools, create or update
`~/.rote/flows/<name>/deps.toml` before replay. This is not optional for
crystallized TypeScript plays: if the play calls `rote.exec({ argv })` for `git`,
`gh`, `cargo`, `python`, `node`, `jq`, `rg`, or any other local executable, the
play directory must include a matching dependency manifest before the play is
marked `released`.

```toml
schema_version = 1

[[tools]]
id = "github-cli"
command = "gh"
required = true
version_requirement = ">=2.0.0"

[files]
required = ["input.txt"]
```

Declare `version_requirement` to enforce a version: preflight probes the tool's
own version flags (`--version`, `-V`, `version`, `-v`) against the located
binary. If the version cannot be inferred, preflight confirms the tool is
present and passes, reporting it as version-unverified rather than failing.
Omit `version_requirement` to check presence only. Preflight never runs
manifest-supplied commands and has no install side effects.

Then run:

```bash
cd ~/.rote/flows/<name>
rote deps check deps.toml
```

Do not auto-install tools into global locations unless the user approved that
specific provisioning policy. A crystallized play should declare dependencies
so the target environment can check or provision them before work begins.

If `rote deps check deps.toml` reports missing required tools, stop and elicit
an install decision from the user before continuing. Show the missing tool,
why the play needs it, and the lowest-risk install scope available. Prefer
project-local or rote-managed installs over global package manager installs.
Only install globally when the user explicitly approves that scope.

Use this shape:

```text
The play needs these missing tools before replay:
- gh: required for GitHub PR and Actions checks

I can install/provision them using one of these scopes:
- rote-managed/project-local: preferred when available; keeps replay isolated
- user/global package manager: broader machine mutation; requires explicit approval

Do you want me to install/provision the missing tools, or should I leave the
play in draft until you install them?
```

After any approved install or user-managed install, rerun:

```bash
cd ~/.rote/flows/<name>
rote deps check deps.toml
```

Never mark the play `released` while required dependencies are still missing.

## Mixed Workflows

For API plus CLI work, use the adapter for typed API calls and `rote proc run` for
local CLI facts. Example shape:

```bash
rote POST /github '{"method":"GET","path":"/repos/$owner/$repo"}' -t -s
rote proc run -- gh repo view "$owner/$repo" --json name,visibility
rote query @1 '.' -r
rote query @2 '.stdout.text' -r
```

The `$owner`/`$repo` above are exploration-time shell/workspace substitutions. In a crystallized
play's `steps:`, `$param` tokens are resolved by the DAG runner from the play's declared
`parameters:` — a different mechanism with the same spelling. Recorded `rote proc run` commands
carry no tokens at all (literal argv), which is why exported drafts need the literals generalized.

For browser plus CLI work, use `rote-browse` for page state and `rote-shell`
for local processing of snapshots/artifacts. Do not drop into raw Playwright or
raw shell when a rote primitive can preserve the evidence.

Adapter, `process.exec`, and typed `browser.navigate|extract|click|type` actions are first-class
`steps:` effects. Default crystallization places them in one effect DAG and gives the presentation
body their completed, checkpoint-restored, failed, skipped, or blocked observations through the
presentation SDK. Browser extracts stay bounded projections, and stateful browser actions keep the
ordering and runtime/auth dependencies recorded by `rote-browse`.

## Crystallization Rule

After a CLI workflow produces the requested result, run the reuse triage before the final
answer — one verified run is enough to classify; a workflow that has already worked twice is
simply a stronger save candidate. Before presenting, run `cd <workspace-path> && rote ls`: it
surfaces the workspace's `@@` state and the `[MANDATORY PROTOCOL]` pending-stub warning when a
reusable result has no pending play yet. The canonical replay command depends on the play's
execution model.

Explicit legacy TypeScript plays with no `steps:` use rote's bundled Deno:

```bash
rote deno run --allow-all ~/.rote/flows/<name>/main.ts
```

Every play containing frontmatter `steps:`, including the default
`steps_with_presentation` artifact, uses the play runner:

```bash
rote play run ~/.rote/flows/<name>/main.ts param=value
```

## Crystallization Workflow

When the user says yes, do the full release discipline:

1. Choose the correct crystallization path:

   - Process-only recorded workspace: use `rote workspace export <path>` with no shape flags. It
     emits the default steps + presentation `process.exec` play without a fake adapter. Then generalize the
     draft: recorded argv is literal, so substitute `$param` tokens for recorded values and prune
     inferred `parameters:` entries that are not part of the contract. Do not run `rote play
     pending save`, `rote play template create`, or `rote play frontmatter`; those commands still
     require a real `--adapter`.
   - Mixed adapter/process/browser workspace: use pending save or direct template/export. Run the
     pending-save command unchanged; it already encodes the steps + presentation shape and recorded dependencies.
     With direct template/export, no shape flags means schema-v1 `steps:` plus presentation. Pass
     only real adapters or browser runtime endpoints to `--adapter`; never pass `process`, `shell`,
     or `adapter/process`.
   - Explicit steps-only request: use `--with-steps` for a template or `--format steps` for export,
     without a presentation flag.
   - Explicit legacy shell/no-steps request, or a named shell capability the step language
     does not support (PTY interaction; background leases/streams with concurrent mid-lease
     work): write `~/.rote/flows/<name>/main.ts` manually with `@rote-frontmatter`, `FlowOutput`, and shell
     SDK calls. This is the legacy escape, not the default.

   Minimal **default steps + presentation** process frontmatter shape — this is what a generalized export
   looks like and the shape to copy for the default path (full worked example:
   `rote guidance typescript play-creation`):

   ```typescript
   /**
    * Repo Changelog
    *
    * Lists recent commit subjects for a local checkout.
    *
    * @rote-frontmatter
    * ---
    * name: repo-changelog
    * description: "Lists recent commit subjects for a local git checkout."
    * provenance:
    *   author: Your Name <you@example.com>
    *   tier: local
    *   created_at: 2026-01-01T00:00:00.000000+00:00
    *   rote_version: 0.53.0            # your current rote --version
    * metadata:
    *   rote_version: 0.53.0            # must parse as a real version
    *   status: draft
    *   kind: atomic
    *   flow_type: sequential
    *   execution_model: steps_with_presentation
    *   format: typescript
    *   requires_endpoints: []
    *   requires_sessions: false
    *   tags:
    *   - shell
    *   - typescript
    * parameters:
    * - name: root
    *   type: string
    *   required: true
    *   description: "Absolute path to the git checkout"
    * - name: count
    *   type: string
    *   required: false
    *   default: "20"
    *   description: "Number of commits to list"
    * steps:
    *   changelog:
    *     type: process.exec
    *     argv: [git, -C, $root, log, --oneline, -n, $count]
    * ---
    */

   const { FlowOutput, isProcessExecBody, loadPresentationContext, stepName } =
     await import("__ROTE_PRESENTATION_SDK__");
   const out = new FlowOutput();
   const ctx = await loadPresentationContext();
   const log = ctx.requireAvailable(stepName("changelog"));
   if (!isProcessExecBody(log.body)) throw new Error("changelog is not a process.exec observation");
   const stdout = log.body.stdout?.text;
   if (stdout === undefined) throw new Error("changelog captured no stdout");
   const lines = stdout.split("\n").filter((l) => l.trim().length > 0);
   out.human(lines.join("\n"));
   out.summary(`${lines.length} commits in ${String(ctx.params.root)}`);
   out.result({ run_id: ctx.run.run_id, commits: lines });
   ```

   Minimal explicit legacy shell-only frontmatter shape (the opt-in escape, NOT the default):

   ```typescript
   /**
    * CSV Sales Report
    *
    * Summarizes a local CSV into JSON and a text report.
    *
    * @rote-frontmatter
    * ---
    * name: csv-sales-report
    * description: "Summarizes a local CSV into JSON and a text report using rote shell primitives."
    * provenance:
    *   author: Your Name <you@example.com>
    *   tier: local
    *   created_at: 2026-01-01T00:00:00.000000+00:00
    *   workspace: csv-json-report-demo
    *   rote_version: 0.49.0
    * metadata:
    *   rote_version: 0.49.0
    *   status: draft
    *   kind: atomic
    *   flow_type: sequential
    *   format: typescript
    *   requires_endpoints: []
    *   requires_sessions: false
    *   parameters:
    *   - name: input
    *     type: string
    *     required: false
    *     default: "input.csv"
    *     description: "Input CSV path"
    *   - name: out_dir
    *     type: string
    *     required: false
    *     default: "out"
    *     description: "Output directory"
    *   tags:
    *   - shell
    *   - process
    *   - typescript
    * ---
    */
   ```

   > **Frontmatter must parse:** a `provenance:` block requires `author`,
   > `tier`, `created_at`, and `rote_version` (only `workspace` is optional),
   > and `metadata:` requires its own `rote_version` — the provenance copy is
   > a separate field, not a substitute; omitting either fails parsing.
   > A missing `author` fails `rote deno run <play>` with a generic
   > "configuration error"; the real cause ("provenance: missing field
   > `author`") shows only in `rote play index --rebuild` stderr. The
   > `created_at` / `rote_version` in the example are placeholders — use a real
   > timestamp and your current `rote --version`.
   >
   > **Surviving `rote play lint`:** lint runs the body in lint mode against a
   > synthetic workspace. Shell SDK reads are lint-aware (`proc.stdout.text()`
   > → `""`, `proc.exit()` → `{kind:"code",code:0}`, `proc.files()` → `[]`), so
   > no `isLintMode()` guard is needed. But: (a) parsing must tolerate empty
   > captured output (e.g. `text.split(...)` on `""`), (b) resolve relative path
   > args against `Deno.cwd()` *before* `Rote.workspace()` switches the cwd, and
   > (c) give parameters defaults — lint runs the play with no arguments.

2. Create `~/.rote/flows/<name>/main.ts`.
3. For default steps + presentation and explicit steps-only plays, put the replay graph in frontmatter `steps:`.
   The default presentation body only reads completed observations through
   `__ROTE_PRESENTATION_SDK__`. For explicit legacy SDK plays, use TypeScript shell primitives for process
   work: `rote.exec`, `rote.execBackground`, `rote.execStatus`,
   `rote.execStop`, `rote.followFile`, `rote.followProcess`, `rote.ptyRun`,
   `rote.depsCheck`, and `rote.execMany`. Use adapter handles from
   `runPreflight(...)` for adapter work.
4. Create `~/.rote/flows/<name>/deps.toml` for every local tool or required
   input file. For shell-only plays, prefer a real `deps.toml` file over
   frontmatter-only dependency prose so replay can run `rote deps check`.
5. Keep frontmatter parameters, usage text, and the flag parser in lockstep.
   If frontmatter names use underscores, the parser must accept those exact
   names or documented aliases; do not ship `github_repo` metadata with only a
   `--github-repo` parser unless both forms work.
6. Start with `metadata.status: draft`. Do not set `released` until dependency
   preflight and replay QA have passed.
7. Test the draft with at least three distinct inputs. For the default steps + presentation play and every other
   play containing `steps:`:

   ```bash
   rote play run ~/.rote/flows/<name>/main.ts param=<input-a>
   rote play run ~/.rote/flows/<name>/main.ts param=<input-b>
   rote play run ~/.rote/flows/<name>/main.ts param=<input-c>
   ```

   Only for an explicit legacy TypeScript body with no `steps:`:

   ```bash
   cd ~/.rote/flows/<name>
   rote deps check deps.toml
   rote deno run --allow-all ~/.rote/flows/<name>/main.ts <input-a>
   rote deno run --allow-all ~/.rote/flows/<name>/main.ts <input-b>
   rote deno run --allow-all ~/.rote/flows/<name>/main.ts <input-c>
   ```

   The DAG runner takes named `key=value` parameters, while the legacy body takes its declared
   positional argument order.

   If dependency preflight fails, ask the user whether to provision missing
   required tools, with the install scope and side effects stated plainly. Do
   not continue release QA until dependency preflight passes or the user says
   to keep the play as `draft`.

8. Use the right QA gate for the execution shape.

   Declarative `steps:` DAG plays:

   ```bash
   rote play validate ~/.rote/flows/<name>/main.ts
   rote play run ~/.rote/flows/<name>/main.ts --dry-run param=value
   rote play run ~/.rote/flows/<name>/main.ts param=value
   rote play run ~/.rote/flows/<name>/main.ts --resume latest param=value
   ```

   `rote play lint` gates `rote play release` for every shape — release runs
   lint and refuses on failure. For pure `steps:` DAG plays the body checks are
   largely vacuous (the DAG runner bypasses the TypeScript body), but the gate
   still applies; for `steps_with_presentation`, lint is the presentation body
   contract check and should run before release.

   Authored SDK plays:

   ```bash
   rote play lint <name>
   rote deno run --allow-all ~/.rote/flows/<name>/main.ts --help
   rote deno run --allow-all ~/.rote/flows/<name>/main.ts <known-good-input>
   ```

   In both shapes, loop until hardcoded paths, missing dependency declarations,
   mismatched parameters, raw shell leaks, and output-format issues are fixed.

9. Mark release only after QA passes. Use `rote play release <name>` for the
   lifecycle transition (it runs the lint gate; do not edit frontmatter
   `status` by hand), then verify search:

   ```bash
   rote play release <name>
   rote play index --rebuild
   rote play search "<name or relevant keywords>"
   ```

10. If a pending workspace stub was used, discard it after release:

   ```bash
   rote play pending discard <workspace>
   ```

Do not claim a shell-derived play is released from memory. The release claim
must point back to `rote deps check deps.toml`, command output, saved responses,
and successful replay commands for the play's execution model.

## Step Reference Rules For Mixed DAGs

In `steps:` plays, `@step{.path}` resolves against the unwrapped step response body, not the
persisted `@N` envelope on disk. For example:

```text
@repo_adapter{.full_name}
```

Presentation code must still tolerate legacy adapter envelope residue when normalizing old cached
observations, but new `@step` references never navigate that envelope. For process steps, the
response body is `process.exec`, so downstream steps can
reference fields such as `.stdout.text` or captured file metadata. Keep
`process.exec` `argv`, stdin paths, and capture paths scalar after resolution.
Do not pass large adapter/browser payloads through argv when a file bridge or a
small scalar projection is clearer.

`$param` tokens resolve in the same fields (`argv` elements, adapter `params:` values, stdin and
capture paths) from the play's declared `parameters:`. An unresolved `$name` passes through as a
literal, and `$item`/`$item_index` are reserved for `for_each` fan-out.
