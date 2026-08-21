---
name: rote-typescript-transformations
description: >
  Author or debug TypeScript transformations for cached rote responses and TypeScript plays. Use for
  `FlowOutput` shape, deterministic response handling, `rote deno` execution, and tests before
  returning to play authoring or workspace work.
---

# rote-typescript-transformations

All `rote-<name>` references in this document — including every name in the Handoff
Contract — are companion **skills**, never CLI commands (`rote-shell` is not `rote shell`).
Invoke them through the runtime's skill mechanism; only literal `rote …` commands run in a
terminal.

Use this skill when cached rote responses need TypeScript transformation or a TypeScript play body is
being authored. `rote grammar deno` and `rote guidance typescript essential` remain the source of
truth for execution syntax and SDK import forms.

This is a helper skill, not a lifecycle owner. Return to the invoking owner after the transformation
is implemented and tested: `rote-flow-authoring` when authoring invoked it, `rote-workspace` when
workspace invoked it. Do not stop here when release, index/search verification, pending cleanup, or
final presentation is still outstanding, and do not hand authoring-owned state back to a substrate
skill.

## Browser Boundary

If the TypeScript body navigates, snapshots, finds, clicks, types, screenshots, restores browser
auth, or controls browser replay, this skill does not own that SDK surface. Run:

```bash
rote guidance browser play-authoring
```

Then return to `rote-flow-authoring`. Do not search the installed SDK barrel to discover the common
browser API before checking this version-matched runtime guide.

## Execution Rules

- Run legacy TypeScript plays, with no frontmatter `steps:` block, through
  `rote deno run --allow-all`.
- Run TypeScript plays with frontmatter `steps:` through `rote play run`. Raw `deno` skips the
  effect plane and provides no presentation input; `rote deno run` reroutes `steps:` plays to the
  play runner, but do not rely on the reroute — use `rote play run` directly.
- Run play files from outside the active workspace.
- Do not call system `deno` directly.
- Do not prefix the binary with `~/.rote/bin/`; use `rote` on `PATH`.

Typical execution:

```bash
rote deno run --allow-all /absolute/path/to/main.ts [args]
```

Use SDK imports exactly as shown by live rote guidance. Avoid npm-style package assumptions unless
the current rote version explicitly supports them.

For `steps_with_presentation`, do not import `sdk/ts/mod.ts` or construct `Rote`. Import
`__ROTE_PRESENTATION_SDK__`, then use `loadPresentationContext()`, literal `stepName("...")`
references, `isProcessExecBody`, `isBrowserBody`, and `FlowOutput`. The context's `ctx.params`
(`Readonly<Record<string, unknown>>`, declared defaults applied) carries the play's invocation
parameters — read plain inputs there instead of echoing them through a step's stdout or touching
`Deno.args`. Template/export also creates or
merges the sibling `deno.json` import mapping for editor type-checking. The `main.ts` source stays
machine-agnostic by importing the virtual specifier. The editor map may use a relative target in the
standard plays layout or an absolute `file://` URL to the local bundled presentation SDK for an
arbitrary/workspace export. That machine-local resolution belongs only in `deno.json`; the runner
supplies its own sandbox import map at execution time. Never hard-code the SDK path or file URL in
`main.ts`.

Before adding package-relative imports or process-payload references, run
`rote guidance typescript play-creation`. Follow that authoritative package contract instead of
inferring module imports or resource addressing from a file's location.

## Normalize Presentation Observations

Treat the presentation body as a schema-agnostic normalizer over recorded facts, not as a second
effect runner. Preserve the SDK's `single`, `fan_out`, and `empty_fan_out` distinction. Normalize
each fan-out item independently; an observed empty fan-out is `[]`, while an absent optional value
stays absent rather than becoming `""`, `0`, `false`, or `[]`.

If a step can be restored, failed, skipped, or blocked, inspect
`ctx.step(stepName("literal")).outcome.status`.

A checkpoint-restored step has status `restored`. It does not have status `skipped`.
When the play runs with `rote play run --resume`, handle `restored` in every status switch.

Use `requireAvailable` when the step must provide output. It accepts completed and restored
steps. It throws for condition-skipped, failed, and blocked steps.

For skipped, failed, or blocked steps, return a typed warning or error. Do not create a successful
payload from an unavailable result. For play inputs, narrow `ctx.params` values explicitly
(`typeof ctx.params.owner === "string"`) rather than coercing absent parameters into fabricated
defaults.

Normalize completed bodies by substrate:

- Narrow `process.exec` with `isProcessExecBody`; validate `status.exit` before consuming optional
  `stdout.text`, `stderr.text`, or file observations.
- Narrow browser bodies with `isBrowserBody`; switch on `op` and retain bounded page/element facts.
- Treat other bodies as adapter observations. Accept direct JSON, plain text, JSON encoded in one
  text block, and the narrowly recognized legacy `{result:{content:[{type:"text",text}]}}`
  residue. Preserve zero or many valid text blocks as an `adapter_text_blocks` observation without
  joining or dropping them. Only non-text/malformed blocks or unexpected values become malformed
  observations with a reason; never guess a payload.

The full concrete normalization pattern lives in `rote guidance typescript play-creation`; copy it
and specialize only the final domain projection.

## Transform Cached Responses

When a transformation can be expressed clearly with jq, prefer `rote query @N '<jq-filter>' -r` and return
to `rote-workspace`. Use TypeScript when the task needs richer validation, grouping, date handling,
joins, cross-response merging, or formatting than jq should carry.

Keep transformations deterministic:

- Treat missing fields explicitly instead of fabricating zero, empty string, or default dates.
- Keep user-provided parameters separate from constants.
- Return structured data plus a concise summary when useful.
- Avoid reading rote workspace files directly; use response IDs, exported fixtures, or SDK helpers.
- Preserve raw adapter responses only when the user or play contract needs them.

## FlowOutput Guidance

Design `FlowOutput` for future agents as well as humans:

- `summary` for the short answer.
- `data` or domain-specific fields for machine-readable results.
- `warnings` for partial, skipped, or best-effort outcomes.
- No secrets, local paths, transient workspace IDs, or one-off response IDs unless requested.
- Stable field names that match documented frontmatter output expectations.

If the transformation is part of a reusable play, return the proposed `FlowOutput` shape to
`rote-flow-authoring` before release.

## Testing Expectations

Test with representative cached data or fixture input before release. Cover no-result,
partial-result, malformed/optional fields, and at least one user-provided edge case. After changes,
rerun legacy TypeScript through `rote deno run --allow-all` and plays with frontmatter `steps:`
through `rote play run`, rather than a standalone TypeScript runner.

If a cached response query fails, inspect the cached response first. If the response is an adapter
error, fix the upstream call, auth, base URL, or arguments before adding transformation workaround
code.

## Return Fields

Return these fields to the invoking owner (`rote-flow-authoring` or `rote-workspace`):

- Input response IDs or fixture sources.
- Transformation path: jq, TypeScript helper, play body, or blocker.
- `FlowOutput` shape and warnings contract.
- Test commands and representative cases covered.
- Remaining assumptions, missing fields, or upstream adapter fixes needed.

## Handoff Contract

- Use when: TypeScript play logic or cached-response transformation is needed for workspace output or
  reusable play authoring.
- Preconditions: input response IDs, fixture data, or play parameters are available; live `rote
  grammar deno` guidance can be checked or its blocker is known.
- Owns: TypeScript execution rules, SDK import guidance, deterministic transformation design,
  `FlowOutput` shape, and transformation-specific test expectations.
- Hands off to: live `rote guidance browser play-authoring` for browser TypeScript;
  `rote-flow-authoring` for play lifecycle/release; `rote-workspace` for additional adapter calls or
  cached response queries; `rote-command-patterns` for command syntax; `rote-troubleshooting` after
  repeated unchanged failures.
- Returns to: the invoking owner (`rote-flow-authoring` or `rote-workspace`) with transformation path,
  output shape, tests, and blockers.
- Stop when: transformation is tested, jq is sufficient and the owner can continue, required data is
  missing, or troubleshooting becomes the correct owner.
- Completion signal: tested transformation or explicit blocker plus the owner skill to resume.
