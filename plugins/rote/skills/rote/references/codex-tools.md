# Codex Tool Mapping

Use this reference when a rote workflow runs inside Codex and sandbox, approval, or filesystem
rules affect the next action.

## Action Map

| Rote skill action | Codex equivalent |
| --- | --- |
| Invoke a companion skill | Load the named skill through the runtime's skill mechanism when available; otherwise follow the installed `SKILL.md` exactly. |
| Run a rote command | Use the shell command tool with one `rote ...` command at a time. |
| Read or edit files | Use the runtime's file tools or patch mechanism; avoid shell file edits when a safer edit tool exists. |
| Dispatch a subagent | If subagents are unavailable, keep ownership in the active skill and preserve the handoff packet in the response or workspace summary. |
| Handle approval prompts | Retry only the blocked command with the required sandbox escalation and a narrow justification. |

## Sandbox Rules

- Installed Codex targets may include rules that allow `rote *` and `~/.rote/flows/*`; still read
  command output before assuming a step succeeded.
- If sandboxing blocks a command, do not widen the request to the whole home directory. Request the
  specific `rote` command, workspace path, or flow path.
- Do not use destructive commands unless the user explicitly requested that destructive action.

## Filesystem Constraints

- Prefer paths returned by `rote init`, `rote workspace inspect`, `rote flow search --json`, or
  `rote flow list --json`.
- When a workflow says to write a handoff summary, write it through the available file-editing tool
  in the named workspace or artifact path.
- If the runtime cannot load another skill directly, continue from the current context but keep the
  target skill name and handoff packet visible.
