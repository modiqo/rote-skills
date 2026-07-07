# Copilot CLI Tool Mapping

Use this reference when rote guidance names generic actions but the current runtime does not expose
Claude Code or Codex tool names.

## Action Map

| Rote skill action | Copilot CLI equivalent |
| --- | --- |
| Invoke a companion skill | Ask the runtime to activate or follow the named installed skill; if direct activation is unavailable, read the installed skill body and treat its contract as authoritative. |
| Run a rote command | Use the CLI command execution facility, one rote command at a time. |
| Inspect files | Use the safest available file-read or search operation before shell commands. |
| Edit files | Use the runtime's patch or edit operation; keep generated edits scoped to the named files. |
| Dispatch a subagent | If there is no subagent primitive, continue inline and record the handoff packet as a resumable checklist. |
| Ask for approval | Name the exact command, path, or network action that needs approval. |

## Limitations

- Do not assume Copilot has Claude-specific tools such as `Task`, `TodoWrite`, or `Bash` by name.
- Do not assume a skill can be activated programmatically; when activation is unavailable, make the
  handoff explicit in prose and follow the target `SKILL.md` content.
- Preserve rote's sequential command rule even if the runtime offers parallel command execution.

## Instruction Files

- Respect repository instructions already loaded by the runtime.
- If a workflow says to update project memory, use the repository's established memory or
  instructions file instead of inventing a Copilot-specific filename.
