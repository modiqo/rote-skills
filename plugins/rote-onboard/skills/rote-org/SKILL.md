---
name: rote-org
description: Manage rote registry organizations — create / delete an org, list and change member roles, invite or remove users, revoke or list pending invites, view plan and usage. Use when the user says things like "create an org", "invite teammate to my org", "list org members", "remove user from org", "change role", "revoke invite", "what's my org usage", or otherwise references organization administration in rote.
---

# rote-org — Organization management

All operations go through `rote registry org <subcommand>`. These commands handle auth, validation, and error rendering.

Login first: `rote login` (auth is shared with the main rote skill).

Roles: `admin` | `developer` | `reader`. Invite default role: `developer`.
Slug = URL-safe org identifier (e.g. `my-org`). `--help` calls this `<org>` for invite/members commands and `<slug>` for the rest — same thing.

## Command reference

| Goal | Command |
|---|---|
| List your orgs | `rote registry org list [--json]` |
| Create org | `rote registry org create --slug <slug> --name "<Name>"` |
| Delete org (owner only) | `rote registry org delete <slug>` |
| Plan, usage, feature flags | `rote registry org usage <slug> [--json]` |
| Invite member | `rote registry org invite <slug> <email> [--role <role>]` |
| Revoke a pending invite | `rote registry org invite revoke <slug> <email>` |
| List pending invites | `rote registry org invite pending <slug> [--json]` |
| List members | `rote registry org members <slug> [--pending] [--json]` |
| Remove a member | `rote registry org members remove <slug> <email>` |
| Change a member's role | `rote registry org members role <slug> <email> <admin\|developer\|reader>` |

`--pending` on `members` appends still-pending invites to the listing.

## Behavior to know

- Invites auto-onboard: invitee gets an email; on signup they become a
  member with the assigned role. There is no separate accept step.
- Role changes and member removals require admin or owner of the target org.
- `delete` is owner-only and irreversible

## Related (push targeting)

Pushing an adapter to an org namespace is not under `org` itself. Instead:

```bash
rote registry adapter push <adapter-path> <slug> [--private]
```

If the user is about to push and wants to check quota first, run
`rote registry org usage <slug>` and report the plan and usage numbers from the response.
