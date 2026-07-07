# Pi Tool Mapping

Use this reference when rote runs in a Pi runtime and the current workflow needs Pi-specific skill
discovery, tool names, or handoff wording.

## Action Map

| Rote skill action | Pi equivalent |
| --- | --- |
| Invoke a companion skill | Use Pi skill discovery for the named `rote-*` skill, then follow that skill's handoff contract. |
| Run a rote command | Use Pi's command execution tool with one command at a time. |
| Read or edit files | Use Pi's file tools; avoid shell reads and writes when file tools are available. |
| Dispatch a subagent | Use Pi's delegation primitive if available; otherwise keep the handoff packet inline. |
| Record a handoff summary | Write the markdown summary at the workspace or artifact path named by the active skill. |
| Ask for approval | Describe the exact command, path, or runtime capability that is blocked. |

## Handoff Language

- Name the target skill explicitly, for example `rote-workspace` or `rote-registry`.
- Include the standard handoff packet when leaving `rote` for a workspace-bound or subagent-capable
  skill.
- If direct skill activation is not available, state that the next section follows the target skill
  contract and keep the return fields visible.

## Discovery Notes

- `rote/SKILL.md` is the active entrypoint for task routing.
- `INDEX.md` is an installed package map, not required active task context.
- Load `skill-workflow-map.md` only when the short routing table is not enough.
