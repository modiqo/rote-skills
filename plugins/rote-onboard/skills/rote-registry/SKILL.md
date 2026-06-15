---
name: rote-registry
description: >
  Share rote artifacts to the registry and manage the org around them. Triggers at the
  schelling-point moment — right after an adapter is first created or a flow is crystallized
  (draft or release) — to check whether the artifact already exists in your orgs, tell you
  whether you need to push, and (after a push) surface org members and offer to invite others
  for review or use. Also runs standalone for `push`, `/show usage` (plan + quota across all
  your orgs), invites, and member management. Use when the user says "push to registry",
  "share my adapter/flow", "publish this", "show my usage / quota", "invite someone to my org",
  "who's in my org", "/rote-registry". Determines every fact from live `rote registry` commands
  — never from memory.
---

# rote-registry — share artifacts, manage the org

Take a freshly-minted adapter or flow and get it into the registry the way the rote-vscode Hub
does it: **check before you push** (don't burn quota re-pushing something already there), push
at the visibility the user chooses, then **turn the push into collaboration** by surfacing org
members and offering invites.

**Follow the shared operating rules in [`../INDEX.md`](../INDEX.md) § "Shared operating
rules"** — on a fresh run, offer the full permissions (both `permissions.allow`
`Bash(rote:*)`/`Bash(cd:*)` AND `permissions.additionalDirectories` `~/.rote`) so the user
isn't prompted on every step.

Core rules:
- **Determine facts from live `rote registry` commands, never from memory.** (If the rote
  binary isn't on PATH, resolve it via the **narrow probe** — check `$HOME/.local/bin/rote`
  then `$HOME/.cargo/bin/rote`, never a deep `find` of the home dir. See INDEX § 1b.)
- **One command per Bash call, strictly sequential — never parallel.** Probes gate decisions.
- **Auth-gate first.** Every registry op needs a valid session — see Stage 0.
- **Existence is checked by fingerprint/version, not just name** — re-pushing an identical
  artifact wastes a quota slot. See Stage 2.
- **Visibility is never silently defaulted.** Public is hard to walk back; always ask.
- Secrets discipline: pushing an adapter ships its *config*, not its token values. Confirm
  the user understands before a **public** push.

---

## Two entry modes

**A. Hand-off (the schelling point).** `rote-adapter-create` (Stage 6) and the flow-crystallize
path invoke this skill right after minting. The artifact id is known; jump to Stage 1 with it.

**B. Standalone.** User invokes `/rote-registry` directly. Present the menu:
- **Push / share** an adapter or flow → Stage 1
- **Show usage** (plan + quota across all orgs) → Stage U
- **Manage org** (members / invites) → Stage 4
- **Find** an artifact in the registry (`adapter|flow search`) → one-off

---

## Stage 0 — Auth gate

```bash
rote registry whoami --verbose
```
- Authenticated → note the email; continue.
- Not authenticated → `rote registry login` (or `rote login --provider google|github`). It's a
  browser flow — tell the user to finish in the browser, then re-run `whoami` before proceeding.

---

## Stage 1 — Which orgs, and is the artifact already there?

**This is where the "hub" concept first appears — give the hub What/Value beat before asking
where to push.** Deliver the **Hub** beat from [`../INDEX.md`](../INDEX.md) § "Primitive
intros" (~3–4 lines: a working flow/adapter shared so the team and community reuse it — the way
a teacher's lesson saves every student rediscovering it; shared artifacts become collective
intelligence. Value: token savings across the org and community, with the same determinism).
*Then* proceed.

List the orgs the user belongs to (the push targets):
```bash
rote registry org list --json
```
Returns the orgs with `slug` / `name`. If the user is in **none**, offer to create one
(`rote registry org create --slug <slug> --name "<name>"`) or hand off to the **rote-org** skill;
without an org there's nowhere to push.

Then run the **existence check** against each candidate org (Stage 2).

---

## Stage 2 — Existence check (do you even need to push?)

For an **adapter** named `<id>`, compare the local fingerprint to what's published per org:

```bash
rote adapter info <id>
```
(grab the `Fingerprint` and `Version` lines from the output), then for each org slug:
```bash
rote registry adapter info <slug>/<id> --json
```
- **Errors / empty** → not published in that org → **push needed**.
- **Returns `{adapter:{fingerprint}, version:{version}}`** → compare:
  - `remote.adapter.fingerprint == local.fingerprint` **and** version matches → **already in
    sync; no push needed.** Say so — don't waste a quota slot re-pushing identical content.
  - fingerprint differs, or local version > remote version → **local is ahead; push to update.**

For a **flow** named `<name>`:
```bash
rote registry flow info <slug>/<name> --json
```
- Errors / empty → **push needed**.
- Present → compare published version to local; if local is newer → **push to update**, else
  **in sync**.

Summarize the per-org verdict plainly, e.g.:
> `linear` — **not in `conikee-home`** (push to share) · **in sync in `modiqo`** (nothing to do).

If every org says "in sync," tell the user there's nothing to push and offer Stage 4
(invite/members) or Stage U (usage) instead.

---

## Stage 3 — Push (only where needed, at chosen visibility)

For each org where a push is warranted, **ask visibility first** (never default it) via
`AskUserQuestion`:
- **Private** — org-only. For review/use inside the org. Reversible-ish.
- **Public** — anyone on the registry can pull it. Confirm the user means it; for a public
  **adapter** push, remind them it ships config (base URL, auth scheme) — not token values, but
  still worth a glance.

**Adapter** (pack + push):
```bash
rote registry adapter publish <id> <slug>
```
Add `--private` for a private push (omit for public).

**Flow** (auto-archives + walks dependencies). Push the flow's `main.ts` path; `--check-deps`
publishes the adapters it depends on first (skipping any already in sync — same fingerprint
logic as Stage 2):
```bash
rote registry flow push <path-to-flow>/main.ts <slug> --check-deps
```
Add `--private` for private.

**Version conflict recovery** — if a push fails with "version already exists" / "bump the
version", offer a semver bump and retry:
```bash
rote adapter bump <id> [--minor|--major]   # default: patch
```
```bash
rote flow bump <path> [--minor|--major]
```
Then re-run the publish/push. Show the conflict error verbatim before offering the bump.

On success, state what landed where (id, org, visibility, version), then go to Stage 4.

---

## Stage 4 — Turn the push into collaboration (members + invite)

Right after a successful push, offer to invite others to the org you just pushed into so they can
review or use the new artifact. The flow is: **ask → snapshot who's already there → collect a set
of emails → dedup → pick role(s) → invite each sequentially → report**.

### 4a — Ask first

Via `AskUserQuestion`: **invite anyone to `<slug>` to review/use `<artifact>`?**
- If no → done; offer Stage U (usage) or exit.
- If yes → continue to 4b.

### 4b — Snapshot existing members AND pending invites (dedup source)

**Before collecting emails**, pull the current roster + outstanding invites in one call so you
can skip anyone already in or already invited. `--pending` appends invites to the member list:
```bash
rote registry org members <slug> --pending --json
```
Returns `{ "members": [{email, role, …}], "pending": [{email, …}] }`. This is owner/admin-gated —
if it errors with a permissions message, the user isn't an admin of that org: say so and skip the
invite offer (only owners/admins can invite). Build a set of `{email → status}` (member /
pending) from `.members[].email` and `.pending[].email` to dedup against in 4d.

(If a build doesn't support `--json` together with `--pending`, fall back to two calls:
`rote registry org members <slug> --json` and `rote registry org members <slug> --pending`.)

### 4c — Collect the set of emails

Ask the user for **one or more** emails to invite (a list is fine — they can paste several).
Don't invite anything yet.

### 4d — Dedup against the snapshot

For each email, compare against the 4b set and bucket it **before** inviting:
- **already a member** → skip, report "already in `<slug>` as `<role>`".
- **already pending** → skip, report "invite already pending".
- **new** → keep for invite.

Only the *new* bucket proceeds. Show the user the skip list so it's clear why some were dropped.

### 4e — Pick role(s), with the admin caveat

The role determines what the invitee can do. Present all three via `AskUserQuestion` (default
**developer**), and either apply one role to the whole batch or let the user set it per email:

| Role | Can do | Use for |
|------|--------|---------|
| **reader** | Pull/use artifacts in the org. Cannot push or manage. | Teammates who only consume flows/adapters. |
| **developer** | reader **+ push/update** adapters and flows. Cannot manage members or org settings. | Collaborators who contribute artifacts. |
| **admin** | developer **+ manage the org**: invite/remove members, change roles, manage artifacts and visibility. | Co-maintainers you fully trust. |

> **Admin caveat — call this out before granting it.** An admin can invite and remove *other*
> members (including downgrading peers), change anyone's role, and delete/re-publish the org's
> artifacts. It's near-total control short of deleting the org itself (owner-only). Grant it only
> to people you'd trust to run the org; for someone who just needs to contribute flows,
> **developer** is the right default. Don't default to admin to "save a step."

**Seat check first.** From Stage U's `limit.members - current.members`, compute remaining seats.
If the new-bucket count exceeds remaining seats, tell the user up front and invite only as many
as fit (or have them upgrade / free a seat) — don't fire invites you know will hit the cap.

### 4f — Invite each sequentially

Invite **one email per command, in sequence** (never batch into a parallel block — each can fail
independently and you report per-email):
```bash
rote registry org invite <slug> <email> --role <admin|developer|reader>
```
**`org invite` has no `--json`** — parse its human output into an outcome and report each plainly:
- *added* (email already registered → in immediately)
- *pending* (unregistered → invite activates on signup)
- *already a member* / *duplicate pending* → note it, skip (shouldn't happen if 4d ran, but
  handle it idempotently)
- *quota_exceeded* (seat limit hit) → stop inviting; report the org is at its member cap.

After the loop, summarize: who was added, who's pending, who was skipped (and why).

To revoke a mistaken pending invite: `rote registry org invite revoke <slug> <email>`.
To change a role later (e.g. walk back an over-grant): `rote registry org members role <slug> <email> <role>`.

---

## Stage U — Show usage (plan + quota across all your orgs)

Loop every org and render a summary so the user knows their consumption:
```bash
rote registry org list --json
```
then for each slug:
```bash
rote registry org usage <slug> --json
```
The shape: `{ plan, current{members, public_adapters, private_adapters, public_flows,
private_flows}, limit{... null = unlimited}, features{audit, verification} }`.

Render a per-org table — for each metric show `current / limit` (render `null` limit as
**∞ / unlimited**), the plan, and which feature flags are on. Flag anything near a non-null
limit (e.g. ≥80%) so the user sees pressure before they hit it. Example row:

> **conikee-home** · plan `org_admin` · members 1/∞ · private adapters 3/∞ · private flows
> 2/∞ · public 0 · features: audit ✓, verification ✓

This is the `/show usage` answer — quota consumption at a glance, no surprises.

---

## Notes

- **Don't re-push identical content.** The Stage 2 fingerprint/version check exists to protect
  quota — surfacing "already in sync" is a feature, not a non-answer.
- **`adapter publish` vs `adapter push`:** `publish` packs then pushes (the normal path);
  `push` uploads an already-packed `.adapt`. Prefer `publish` for a freshly-minted adapter.
- **Flow drafts vs releases:** the crystallize hand-off fires for both. A draft push shares
  work-in-progress for review; a release push shares the finished flow. Same commands; the
  user's intent (review vs use) just shapes the invite framing in Stage 4.
- For deeper org administration (create/delete orgs, change roles, remove members, manage
  pending invites at length), hand off to the **rote-org** skill — this skill covers the
  push-time slice (members + invite); rote-org is the full admin surface.

---

## Closing line + related skills

**Closing line** (only after a clean push): one dry one-liner keyed to what landed — e.g. the
artifact name + org + that it's now shareable without a middleman registry tax. Shared
convention and rules in [INDEX.md](../INDEX.md). Skip it if the push errored or there was
nothing to push.

**Related onboard skills** ([INDEX.md](../INDEX.md) is the full map):
- **Invoked by:** `/rote-onboard:rote-adapter-create` (after minting an adapter) and the flow
  crystallize path.
- **Full org admin:** the **rote-org** skill · **Tune the adapter first:**
  `/rote-onboard:rote-adapter-config`
