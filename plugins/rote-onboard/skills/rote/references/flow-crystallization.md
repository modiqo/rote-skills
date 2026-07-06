# Flow Crystallization

Use this reference after workspace execution produces a result that may be worth saving as a
reusable flow. The pending lifecycle is mandatory for reusable workspace results: write the pending
stub first, then run or record the pending save command before any scaffold or release work.

If the user already asked to save, release, or make the workflow reusable, treat that request as the
save approval. Do not ask again, but do not skip `rote flow pending write` or
`rote flow pending save`.

## 1. Write the pending stub before final output

Before presenting the final answer, create a pending flow stub that captures the repeatable steps,
inputs, and outputs from the workspace run. Use rote's pending-flow command surface for the current
syntax; if unsure, run `rote grammar export` or `rote guidance agent essential`.

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

For long-running work, after compaction, or before any handoff, confirm the stub is recoverable:

```bash
rote flow pending list
```

The stub should preserve:

- The user's original intent.
- The adapter calls and cached response transformations that produced the result.
- Parameter candidates and any values that should not be hard-coded.
- A concise result shape that a future flow can return.

## 2. Generate the pending save command before final output

After the stub exists, run `rote flow pending save` before answering the user:

```bash
rote flow pending save <workspace-name>
```

`pending save` prints the pre-filled `rote flow template create ...` command; it does not create the
flow by itself. Capture the emitted scaffold command so the save path is recoverable before context
can shift, compact, or hand off.

If the session was interrupted after writing the stub, recover with:

```bash
rote flow pending list
```

Then inspect the relevant pending workspace and rerun `rote flow pending save <workspace-name>` to
recover the scaffold command before continuing the save decision.

## 3. Present results and ask once if save is not pre-approved

Only after pending write and pending save have both completed, present the task result and ask one
explicit yes/no question about saving the workflow when the user did not already request a saved or
released reusable flow.

Use this shape:

```text
Result: <brief task result>

I can save this as a reusable rote flow for next time. Save it? (yes/no)
```

Do not combine the save question with unrelated follow-ups. Do not assume consent from silence,
thanks, or a new task. A direct instruction such as "save this", "release it", or "make it reusable"
is already consent for the save path, so continue after pending save instead of stopping to ask.

## 4. If the user says yes or already asked to save

Run the captured scaffold command from `pending save`, then follow the test/release/index path
surfaced by rote. For TypeScript or export details, prefer live guidance (`rote guidance typescript
essential`, `rote grammar deno`, `rote grammar export`) until the authoring reference is loaded in a
later branch.

Keep the lifecycle distinct:

```text
pending write -> pending save -> flow template create -> implement/test -> flow lint -> flow release -> flow index --rebuild -> flow search
```

`pending save` is not release, a draft flow is not discoverable by normal search, and a successful
test run is not a lifecycle transition. If any step fails, fix the cause before rerunning the same
command.

For requests that already asked to save and release locally, continue through the scaffold/release
path without another approval question, rebuild or refresh the flow index as directed by rote, then
verify discoverability:

```bash
rote flow index --rebuild
rote flow search "<flow-name-or-intent>"
rote flow search "<flow-name-or-intent>" --json
```

If the verification requires an execution check, run the released flow from outside the active
workspace using [flow-search-and-run.md](./flow-search-and-run.md).

## 5. If the user says no

Discard the pending stub through rote rather than deleting files directly:

```bash
rote flow pending discard <workspace-name>
```

Then acknowledge that the workflow was not saved.

## 6. If the answer is unclear or interrupted

If the user gives anything other than a clear yes or no, keep the pending stub and ask for a yes/no
decision again. If the session resumes later, use `rote flow pending list` to recover the stub and
continue from the save question.
