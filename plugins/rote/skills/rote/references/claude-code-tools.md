# Claude Code Tool Mapping

Use this reference only when the active rote workflow needs Claude Code-specific tool names,
permissions, or subagent behavior.

## Action Map

| Rote skill action | Claude Code equivalent |
| --- | --- |
| Invoke a companion skill | Use the skill-loading mechanism for the named skill, then follow that `SKILL.md`. |
| Run a rote command | Use `Bash` with one `rote ...` command at a time. |
| Read or inspect a source file | Use `Read`, `Glob`, or `Grep` rather than shell file-reading commands. |
| Edit a source file | Use `Edit`, `Write`, or `apply_patch` as appropriate. |
| Track multi-step work | Use `TodoWrite` when the task has multiple meaningful steps. |
| Dispatch a subagent | Use the Agent/Task tool when available, passing the handoff packet and expected return fields. |
| Ask for approval | Surface the exact command, directory, or permission the runtime blocked. |

## Rote Command Discipline

- Run dependent rote commands sequentially; do not batch `rote play search`, `rote explore`, and
  cached-response queries in parallel.
- Prefer the `workdir` tool parameter over `cd ... && ...` unless a play reference explicitly says a
  compound command is one logical execution step.
- If a command needs permissions, request the smallest approval that covers the current rote command
  or play path.

## Filesystem Expectations

- Use the absolute workspace path printed by rote when entering `${ROTE_HOME:-$HOME/.rote}`.
- Do not recursively search the home directory for rote; check the known install paths described by
  the active setup or workspace skill.
- Keep platform-specific permission changes outside skill markdown unless a rote command explicitly
  writes them.
