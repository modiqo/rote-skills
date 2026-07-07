---
name: rote-org
description: Manage rote registry organizations — create / delete an org, list and change member roles, invite or remove users, revoke or list pending invites, view plan and usage, and manage org SSO. Use when the user says things like "create an org", "invite teammate to my org", "list org members", "remove user from org", "change role", "revoke invite", "what's my org usage", "set up SSO", "verify an SSO domain", or otherwise references organization administration in rote.
---

# rote-org — Organization management

All `rote-<name>` references in this document — including every name in the Handoff
Contract — are companion **skills**, never CLI commands (`rote-shell` is not `rote shell`).
Invoke them through the runtime's skill mechanism; only literal `rote …` commands run in a
terminal.

All operations go through `rote registry org <subcommand>`. These commands handle auth, validation, and error rendering.

Login first: `rote login` (auth is shared with the main rote skill).

Run one org command at a time, read the result before deciding the next action, and never assume
membership, role, quota, or pending-invite state from memory. Use
[`../rote/references/skill-workflow-map.md`](../rote/references/skill-workflow-map.md) only when a
caller needs the full companion graph.

Roles: `admin` | `developer` | `reader`. Invite default role: `developer`.
Slug = URL-safe org identifier (e.g. `my-org`). `--help` calls this `<org>` for invite/members commands and `<slug>` for the rest — same thing.

## Handoff Contract

- Use when: the user asks to create, delete, inspect, invite to, remove from, change roles in, or
  view plan/usage for a rote registry organization.
- Preconditions: the user is authenticated or login is surfaced as the blocker; target org slug and
  requested admin operation are known; destructive deletes and privileged role changes are explicitly
  confirmed.
- Owns: standalone org administration, member and invite management, role changes, pending-invite
  revocation, plan/usage reporting, and permission/quota error reporting.
- Hands off to: `rote-registry` when org administration was requested as part of artifact push/share
  or when the user returns to registry publication after org setup.
- Returns to: `rote-registry` or the user with org slug, operation run, before/after member or invite
  state, role/quota result, permission blocker, and next recommended registry action.
- Stop when: the org operation completes, auth/permission/quota blocks, destructive confirmation is
  missing, or registry publication should resume.
- Completion signal: org command result summarized from live output with affected org, affected user
  or quota metric, and any follow-up registry handoff.

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
| Show SSO status | `rote registry org sso show <slug> [--json]` |
| Verify SSO domain | `rote registry org sso verify <slug> <domain> [--check] [--json]` |
| Enable / disable SSO enforcement | `rote registry org sso enable <slug> <domain> [--json]` / `disable <slug> <domain> [--json]` |
| Unbind SSO domain | `rote registry org sso unbind <slug> <domain> [--json]` |
| Revoke SSO sessions | `rote registry org sso revoke-user <slug> --email <addr> [--json]` / `revoke-domain <slug> <domain> [--json]` |

`--pending` on `members` appends still-pending invites to the listing.

## SSO

Org SSO is self-service domain management for SAML-backed organizations after
an operator grants the org a provider. Domain verification is two-step:
`verify` prints a DNS TXT record to publish, then `verify --check` re-checks
DNS and completes the verification. Owners can then enable or disable
enforcement, unbind domains, and revoke sessions for cutover or offboarding.
`show` is available to owners/admins; `revoke-user` is owner/admin; domain
verification, enforcement, unbind, and domain-wide revocation are owner-only.

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
