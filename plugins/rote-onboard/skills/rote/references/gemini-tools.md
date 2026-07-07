# Gemini Tool Mapping

Use this reference when a rote workflow runs in Gemini and the next step depends on tool names,
instruction files, or shell/file operation behavior.

## Action Map

| Rote skill action | Gemini equivalent |
| --- | --- |
| Invoke a companion skill | Use Gemini's skill or instruction inclusion mechanism for the named skill when available. |
| Run a rote command | Use the shell execution tool with a single `rote ...` command, then read the complete result. |
| Read or search files | Use Gemini's file and search tools before shell-based inspection. |
| Edit files | Use the patch/edit mechanism and keep edits scoped to the active workflow. |
| Dispatch a subagent | If unavailable, keep the handoff packet in the active conversation and continue inline. |
| Ask for approval | Report the blocked command or file path and the minimum access needed. |

## Instruction-File Expectations

- Respect loaded repository instruction files before applying rote-specific guidance.
- If Gemini uses a runtime instruction file to include skill bootstrap content, keep `rote/SKILL.md`
  as the active orchestrator and treat `INDEX.md` as package documentation.
- Platform references are supporting context; load only the reference needed for the current runtime
  issue.

## Command Discipline

- Preserve rote's one-command-at-a-time rule even when the runtime can launch multiple shell calls.
- Prefer rote-reported absolute paths for workspaces and flow directories.
- If a command fails unchanged twice, route to troubleshooting instead of repeating it.
