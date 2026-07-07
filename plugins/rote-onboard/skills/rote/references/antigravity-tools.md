# Antigravity Tool Mapping

Use this reference when rote runs in Antigravity and tool names, approvals, or workspace behavior
affect the next step.

## Action Map

| Rote skill action | Antigravity equivalent |
| --- | --- |
| Invoke a companion skill | Activate or follow the named installed skill through Antigravity's skill mechanism. |
| Run a rote command | Use the command execution tool with one `rote ...` command at a time. |
| Inspect files | Use the available file-read and search tools before shell-based inspection. |
| Edit files | Use the patch/edit operation; keep edits limited to files named by the active workflow. |
| Dispatch a subagent | Use Antigravity's agent delegation if available; otherwise continue inline with the packet. |
| Ask for approval | Request only the blocked command, workspace path, flow path, or network action. |

## Runtime Rules

- Keep rote command chains sequential. Parallel command execution can violate cached-response and
  workspace dependencies.
- Treat command output as authoritative. Read `@@status`, `@@next`, `@@mandatory`, warnings, and
  cached response IDs before continuing.
- Use absolute paths returned by rote when entering workspaces or flow directories.

## Skill Context

- The active route begins at `rote/SKILL.md`.
- Platform references translate actions into runtime tools; they do not replace the owning skill's
  handoff contract.
- If Antigravity cannot directly activate a named skill, follow that skill's installed `SKILL.md`
  and keep the standard handoff packet in the conversation.
