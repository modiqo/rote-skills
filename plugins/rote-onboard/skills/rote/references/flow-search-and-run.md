# Flow Search and Run

Use this reference when `rote flow search "<intent>"` returns a usable existing flow. When the
matched flow fully covers the request, it resolves the task branch: stop exploring, do not probe
adapters, and do not rebuild the workflow from scratch. When the flow covers only a baseline or
partial result, run or preserve that baseline before routing the uncovered work; do not discard the
flow output while building the combined result.

## 1. Get the path and parameter contract

When the flow name is known (often from a prior `rote flow search`), look it up directly — by bare
name, `<org>/<name>` id, or a path to the flow file. Do not re-filter a search array client-side to
isolate one flow:

```bash
rote flow info <name> --json
```

`rote flow info` is the exact-name counterpart to `rote flow search`: search returns a ranked array
of fuzzy matches for discovery; info resolves a single flow and `--json` emits one object, not a
list. Use the object's fields directly:

- `path` is the absolute flow file path; use it verbatim.
- `parameters` is the ordered positional argument contract.
- Required parameters need values from the user's intent or a targeted follow-up question.
- Optional parameters use their defaults unless the user gave a value.
- The object also carries `id` (org-qualified as `<org>/<name>` for registry-pulled flows),
  `description`, `format`, `source`, `scheme`, `is_composite`, status/kind/flow_type, endpoint and
  session requirements, and adapter bindings when present.

Resolve by identity, and handle the failure modes:

- `rote flow info acme/<name> --json` selects an org flow. A bare name prefers a local flow when
  one exists; otherwise it resolves a single matching flow by name. An ambiguous bare name exits
  non-zero and lists disambiguation candidates — use the qualified id or the full file path (the
  absolute `path` a prior lookup reported).
- If the argument is an existing flow file path, `rote flow info` resolves that path before any
  name/id lookup.
- An unknown name exits non-zero with did-you-mean suggestions from the flow search index.
- `-d, --dir <path>` resolves from an alternate flows root.

If the name is not yet known, discover first with `rote flow search "<intent>" --json`, then look
the chosen flow up by name.

Do not construct `~/.rote/flows/<org>/<name>/main.ts` by hand. Registry-pulled flows include an
organization segment, while local draft flows can have a different shape.

## 2. Map arguments positionally

Flow parameters are positional. Map the user's request to the declared order from `parameters`.
Do not pass `key=value` pairs unless the flow's own documentation explicitly expects them.

Example: if frontmatter declares `start_date`, then `end_date`, then optional `providers`, a May
rideshare receipt task becomes:

```bash
rote deno run --allow-all /path/to/main.ts 2026/05/01 2026/05/31
```

## 3. Run from `/tmp`

Run TypeScript flows with rote's bundled Deno from `/tmp`:

```bash
cd /tmp && rote deno run --allow-all /absolute/path/to/main.ts [args in declared order]
```

Run shell flows directly from `/tmp`:

```bash
cd /tmp && /absolute/path/to/main.sh [args]
```

The `cd /tmp && ...` compound is one logical step. It keeps flow-created workspaces outside the
current workspace directory.

## 4. TypeScript execution rules

- Use `rote deno run --allow-all` for `.ts` flows.
- Do not run TypeScript flow files directly.
- Do not use system `deno`; rote manages its own Deno runtime.
- Do not prefix the `rote` binary with `~/.rote/bin/`; `rote` itself should be on `PATH`.

## 5. Treat matched-flow output as the answer

After a fully matched flow runs, verify the requested output artifact exists and contains the flow's
result, then answer the user. Do not overwrite, rewrite, reformat, enrich, or replace that artifact
with custom research or direct API output unless the user explicitly asks to edit the flow or create
a separate enhanced artifact.

Do not inspect the flow implementation source just because a lookup returned a path. Once
`rote flow info <name> --json` exposes a runnable path and parameter contract, run the flow and
verify the user-visible result. Read source only when the user asks to modify/debug the flow or
live rote guidance says the contract cannot be determined otherwise.

Verification should check the artifact content, not only file existence. Confirm the requested path,
key headings or markers, required parameters such as city/date/output path, and any live-data section
the user requested.

If the user asks for a combined workflow and the existing flow supplies only one part, keep the
baseline flow output intact in the combined artifact. Add only the uncovered content through the
routing path, and preserve any headings, markers, or summary text the baseline flow produced.

## 6. Optional model tracking with `rote run`

Use `rote run` only when the task needs model tracking and cached workspace responses for the
flow execution:

```bash
rote init <workspace> --seq
cd ${ROTE_HOME:-$HOME/.rote}/rote/workspaces/<workspace>
rote model set <model> --provider <provider>
rote run --inference-id $(uuidgen) \
  --model <model> \
  --model-type chat \
  --model-version <version> \
  /absolute/path/to/main.ts [args in declared order]
rote @1 '.result' -r
```

Required tracking fields are `--inference-id`, `--model`, `--model-type`, and `--model-version`.

## 7. Fallback for older rote versions

If `rote flow search --json` is unavailable, resolve the flow from rote's flow listing and inspect
only the flow's frontmatter for parameters. Prefer upgrading rote or using live `rote grammar`
guidance over filesystem searches.
