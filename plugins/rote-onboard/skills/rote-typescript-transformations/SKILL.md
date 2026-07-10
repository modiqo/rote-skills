---
name: rote-typescript-transformations
description: >
  Author or debug TypeScript transformations for cached rote responses and TypeScript flows. Use for
  `FlowOutput` shape, deterministic response handling, `rote deno` execution, and tests before
  returning to flow authoring or workspace work.
---

# rote-typescript-transformations

All `rote-<name>` references in this document — including every name in the Handoff
Contract — are companion **skills**, never CLI commands (`rote-shell` is not `rote shell`).
Invoke them through the runtime's skill mechanism; only literal `rote …` commands run in a
terminal.

Use this skill when cached rote responses need TypeScript transformation or a TypeScript flow body is
being authored. `rote grammar deno` and `rote guidance typescript essential` remain the source of
truth for execution syntax and SDK import forms.

This is a helper skill, not a lifecycle owner. Return to `rote-flow-authoring` or `rote-workspace`
after the transformation is implemented and tested; do not stop here when release, index/search
verification, pending cleanup, or final presentation is still outstanding.

## Execution Rules

- Run legacy TypeScript flows, with no frontmatter `steps:` block, through
  `rote deno run --allow-all`.
- Run TypeScript flows with frontmatter `steps:` through `rote flow run`. Raw `deno` skips the
  effect plane and provides no presentation input; `rote deno run` reroutes `steps:` flows to the
  flow runner, but do not rely on the reroute — use `rote flow run` directly.
- Run flow files from outside the active workspace.
- Do not call system `deno` directly.
- Do not prefix the binary with `~/.rote/bin/`; use `rote` on `PATH`.

Typical execution:

```bash
rote deno run --allow-all /absolute/path/to/main.ts [args]
```

Use SDK imports exactly as shown by live rote guidance. Avoid npm-style package assumptions unless
the current rote version explicitly supports them.

## Transform Cached Responses

When a transformation can be expressed clearly with jq, prefer `rote query @N '<jq-filter>' -r` and return
to `rote-workspace`. Use TypeScript when the task needs richer validation, grouping, date handling,
joins, cross-response merging, or formatting than jq should carry.

Keep transformations deterministic:

- Treat missing fields explicitly instead of fabricating zero, empty string, or default dates.
- Keep user-provided parameters separate from constants.
- Return structured data plus a concise summary when useful.
- Avoid reading rote workspace files directly; use response IDs, exported fixtures, or SDK helpers.
- Preserve raw adapter responses only when the user or flow contract needs them.

## FlowOutput Guidance

Design `FlowOutput` for future agents as well as humans:

- `summary` for the short answer.
- `data` or domain-specific fields for machine-readable results.
- `warnings` for partial, skipped, or best-effort outcomes.
- No secrets, local paths, transient workspace IDs, or one-off response IDs unless requested.
- Stable field names that match documented frontmatter output expectations.

If the transformation is part of a reusable flow, return the proposed `FlowOutput` shape to
`rote-flow-authoring` before release.

## Testing Expectations

Test with representative cached data or fixture input before release. Cover no-result,
partial-result, malformed/optional fields, and at least one user-provided edge case. After changes,
rerun legacy TypeScript through `rote deno run --allow-all` and flows with frontmatter `steps:`
through `rote flow run`, rather than a standalone TypeScript runner.

If a cached response query fails, inspect the cached response first. If the response is an adapter
error, fix the upstream call, auth, base URL, or arguments before adding transformation workaround
code.

## Return Fields

Return these fields to `rote-flow-authoring` or `rote-workspace`:

- Input response IDs or fixture sources.
- Transformation path: jq, TypeScript helper, flow body, or blocker.
- `FlowOutput` shape and warnings contract.
- Test commands and representative cases covered.
- Remaining assumptions, missing fields, or upstream adapter fixes needed.

## Handoff Contract

- Use when: TypeScript flow logic or cached-response transformation is needed for workspace output or
  reusable flow authoring.
- Preconditions: input response IDs, fixture data, or flow parameters are available; live `rote
  grammar deno` guidance can be checked or its blocker is known.
- Owns: TypeScript execution rules, SDK import guidance, deterministic transformation design,
  `FlowOutput` shape, and transformation-specific test expectations.
- Hands off to: `rote-flow-authoring` for flow lifecycle/release; `rote-workspace` for additional
  adapter calls or cached response queries; `rote-command-patterns` for command syntax; `rote-troubleshooting`
  after repeated unchanged failures.
- Returns to: `rote-flow-authoring` or `rote-workspace` with transformation path, output shape, tests,
  and blockers.
- Stop when: transformation is tested, jq is sufficient and the owner can continue, required data is
  missing, or troubleshooting becomes the correct owner.
- Completion signal: tested transformation or explicit blocker plus the owner skill to resume.
