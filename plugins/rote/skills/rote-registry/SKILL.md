---
name: rote-registry
description: >
  Share rote artifacts to the registry and manage the org around them. Triggers at the
  schelling-point moment — right after an adapter is first created or a play is crystallized
  (draft or release) — to check whether the artifact already exists in your orgs, tell you
  whether you need to push, and (after a push) surface org members and offer to invite others
  for review or use. Also runs standalone for push/share requests, visibility changes, usage/quotas
  (plan + quota across all your orgs), invites, member management, and published Play discovery.
  Use when the user says
  "push to registry", "share my adapter/play", "publish this", "make this public/private",
  "change visibility", "find a published play", "show my usage / quota", "invite someone to my
  org", or "who's in my org".
  Determines every fact from live `rote` commands — never from memory.
---

# rote-registry — share artifacts, manage the org

All `rote-<name>` references in this document — including every name in the Handoff
Contract — are companion **skills**, never CLI commands (`rote-shell` is not `rote shell`).
Invoke them through the runtime's skill mechanism; only literal `rote …` commands run in a
terminal.

Take a freshly-minted adapter or play and get it into the registry with the same discipline as
rote's guided setup: **check before you push** (don't burn quota re-pushing something already there), push
at the visibility the user chooses, then **turn the push into collaboration** by surfacing org
members and offering invites.

Use the base `rote` companion skill when the full workflow graph or standard packet shape is needed;
this skill contains the active registry rules. On a fresh run, clear command/filesystem access if the current environment requires it.

Core rules:
- **Determine facts from live `rote registry` commands, never from memory.** (If the rote
  binary isn't on PATH, resolve it via the **narrow probe** — check `$HOME/.local/bin/rote`
  then `$HOME/.cargo/bin/rote`, never a deep home-directory search.)
- **One command at a time, strictly sequential — never parallel.** Probes gate decisions.
- **Auth-gate every `rote registry` command, reads included.** The one anonymous-capable discovery
  command is `rote play search --source registry` — not a `rote registry` command: public scope
  needs no session, accessible scope uses a stored one when usable and reports public otherwise.
- **Existence is checked by fingerprint/version, not just name** — re-pushing an identical
  artifact wastes a quota slot. See Stage 2.
- **Visibility is never silently defaulted on push.** Confirm before a public push; published
  visibility can be flipped later, but already-pulled local copies are unaffected.
- Secrets discipline: pushing an adapter ships its *config*, not its token values. Confirm
  the user understands before a **public** push.

## Handoff Contract

- Use when: a newly created adapter or crystallized play reaches the share point; `rote-flow-run`
  returns an uninstalled, behavior-changing registry result that needs exact-version pull; or the
  user asks for registry push/share, a published artifact visibility change, usage/quota, invite,
  member, artifact search, or registry collaboration.
- Preconditions: `rote play search --source registry` needs only an intent query; every
  `rote registry` command has authenticated through `rote registry whoami --verbose` or surfaced
  the login blocker; artifact id/path and target owner can be elicited; visibility is confirmed
  before any push or in-place visibility change. A fork handoff from `rote-flow-run` supplies an
  exact pinned Play reference and `installation_state: uninstalled`; pull still requires inspection
  and approval for that exact version. A direct `rote-flow-authoring` return supplies the same
  pinned reference and installation state.
- Owns: registry auth gate, org/owner selection, existence checks, dry-run publish/push, visibility
  selection and in-place changes, conflict recovery, usage reporting, and invite-at-share-time
  collaboration. It also owns pulling an exact published Play before a fork handoff.
- Hands off to: `rote-org` for deeper organization administration; `rote-adapter-create` when the
  artifact is not ready to publish; `rote-flow-crystallization` or `rote-flow-authoring` when a play
  must be saved/released before push; `rote-flow-authoring` after an exact Play version is pulled for
  a behavior-changing fork; `rote` for day-to-day routing after sharing.
- Returns to: the caller with artifact kind/id/path, target org/owner, visibility or old/new
  visibility plus changed/no-op status, dry-run verdict, push result or skip reason,
  version/conflict state, and — for a published play — the whole publication result rather than a URI
  alone (produced by *Stage P*, below): play location, verified Play URI, bootstrap URI, resolved run
  reference, execution readiness, execution variant, blockers, published-reference
  execution-verification status and evidence, plus access guidance (resolution and execution
  audiences). A fork handoff also returns the exact pull command, installed identity, local path, and
  successful managed-install state. Then invite results and next recommended skill. A caller given
  only the URI cannot tell whether the reference it received is executable or verified, which is the
  question that stage exists to answer.
- Stop when: the artifact is shared, confirmed in sync, or has the requested visibility; auth/org
  permission blocks, visibility is unconfirmed, quota blocks the requested write, an exact-version
  pull fails, or the owning creation/authoring skill must resume.
- Completion signal: registry state verified from live commands, push/share/visibility/invite result
  summarized, and return data includes the artifact, owner, visibility, a published play's verified
  Play URI and sharing guidance, any completed managed install needed for a fork, and any remaining
  blocker.

---

## Two entry modes

**A. Hand-off (the schelling point).** `rote-adapter-create` (Stage 6) and the play-crystallize
path invoke this skill right after minting. The artifact id is known; jump to Stage 1 with it.

**B. Standalone.** User asks for registry help directly. Present the menu:
- **Push / share** an adapter or play → Stage 1
- **Change visibility** of a published adapter or play → Stage V
- **Show usage** (plan + quota across all orgs) → Stage U
- **Manage org** (members / invites) → Stage 4
- **Find a published Play** — the user asked what exists in the registry → run
  `rote play search "<intent>" --source registry` (no Stage 0 needed); use `--scope public` only
  when identity-independent output is required, then return after the one-off result
- **Find a Play that does X** — a capability request that happens to arrive here → return to the
  main `rote` skill and complete its local-then-registry Play-search gate. An installed Play that
  already covers the request must not be re-resolved from the registry
- **Find an adapter** → Stage 0, then `rote registry adapter search <query>` — it is auth-gated
  like every other `rote registry` read — and return after the one-off result

---

## Stage 0 — Auth gate for every `rote registry` command

```bash
rote registry whoami --verbose
```
- Authenticated → note the email; continue.
- Not authenticated → `rote login`. OAuth opens a browser — tell the user to finish there. Where no
  browser is reachable, `rote login --otp --otp-email <addr>` mails a code and
  `echo <code> | rote login --otp --otp-email <addr> --otp-verify` submits it. Re-run `whoami`
  before proceeding either way.

---

## Stage 1 — Choose the namespace

Use the confirmed personal handle from account information and authorized organizations from:

```bash
rote registry org list --json
```

Resolve the user's choice: **on your handle, or in which organization?** Reuse a destination already
selected in this task. A user without organizations can publish under their confirmed personal
handle. If that handle is missing, claim it through the live registry account commands before
publishing. Create an organization through `rote-org` only when the user chooses a new organization.
Then check the selected namespace in Stage 2.

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

For a **play** named `<name>`:
```bash
rote registry play info <slug>/<name> --json
```
- Errors / empty → **push needed**.
- Present → compare published version to local; if local is newer → **push to update**, else
  **in sync**.

Summarize the per-org verdict plainly, e.g.:
> `linear` — **not in `conikee-home`** (push to share) · **in sync in `modiqo`** (nothing to do).

If every org says "in sync," tell the user there's nothing to push. For a play, continue to Stage P
for each selected owner. Stage 4 applies only to organization-owned artifacts.

---

## Stage 3 — Push (only where needed, at chosen visibility)

For the selected namespace, resolve **public or private** before pushing. Reuse visibility already
authorized in this task; ask only when it is missing.
- **Private** — the owning user can access it under a personal handle; authorized members can
  access it under an organization. Making it private does not revoke already-pulled local copies.
- **Public** — anyone on the registry can pull it. The user must choose public; for a public
  **adapter** push, remind them it ships config (base URL, auth scheme) — not token values, but
  still worth a glance.

**Order matters: push the adapters first, then the play.** A play push verifies every adapter it
depends on is already in the target namespace and hard-fails if one is missing — so the adapters
have to land first.

Before sharing a play, inspect each selected adapter's auth declaration:

```bash
rote adapter info <id> --json
```

Use the selected release's auth declaration for credential setup. Catalog vendor links are optional and never block publication. Keep credential values local.

Both push commands accept `--dry-run`: it runs the full preflight (eligibility, visibility,
version-conflict prediction, and — for plays — dependency reachability) and reports
`would-create` / `would-push` / `would-skip` **without writing anything**. Use it to verify the
artifact and report the status to the user, then re-run the *same command without `--dry-run`* to
actually push to the hub.

**Adapter** (pack + push):
```bash
rote registry adapter publish <id> <slug> --dry-run   # verify + report; writes nothing
rote registry adapter publish <id> <slug>             # push for real
```
Add `--private` for a private push (omit for public).

**Play** — complete the authorized publication using `rote grammar play` and `rote grammar registry`.
Release and create a numbered snapshot when needed, using Rote's returned operation reference or
preparation action. Push the returned snapshot target with the selected namespace and visibility.
Use the live command's dry run before the registry write. Local snapshot creation is not publication.

If the target namespace needs a local copy, let the publication commands prepare it. Fork only when
an editable copy of a managed source is needed. These mechanics do not require separate approval
once the user has chosen the outcome. On a version conflict, apply the normal patch bump and retry
preflight; ask only if the user must choose a different version policy or outcome.

On success, state what landed where (id, namespace, visibility, version). For a play, continue to Stage P;
for an organization-owned adapter, go to Stage 4. Personal publication is complete.

---

## Stage P — Present the published Play URI and verify execution readiness

After the non-`--dry-run` play push succeeds, take `play_uri`, `bootstrap_uri`,
`play_reference`, `play_location`, `play_run_eligible`, `play_run_variant`, and any
`play_run_blockers` from its typed result. When Stage 2 confirms the selected published version is
already in sync, use the same typed fields from that result.

When `play_location` is `registry_only`, the configured registry has no canonical Play host:
`play_uri` and `bootstrap_uri` are intentionally absent. Do not call canonical `play inspect` or
`play run` for that reference. Report that canonical execution verification is not applicable,
pull the pinned artifact, and use the compatible local runner returned by the push result. Preserve
that outcome as `execution_verification: not_applicable` with the reason and exact local command.

For `play_location: canonical`, the pinned `play_uri` is the shareable, disclosure-only Play URI.
Open it first: its JSON or HTML transparency card describes typed inputs and defaults, exact
adapter releases, credential acquisition, effects, distribution integrity, and the ordered
preparation guide. A GET never installs or runs anything. `bootstrap_uri` is a separately
advertised, confirmation-gated transition for a recipient who intends to run the Play and does not
yet have rote. Do not replace the Play URI with a `curl ... | sh` command when sharing.

Inspect the exact execution reference:

```bash
rote play inspect <play_uri> --json
```

Read `data.play_inspect.reference` and `data.play_inspect.execution` from the successful output.
Present `play_uri` as the URI to share, `bootstrap_uri` as an advertised transition rather than the
identity of the Play, and `data.play_inspect.reference` separately as the resolved `rote play run`
reference. Only recommend following the bootstrap transition when
`data.play_inspect.execution.play_run_eligible` is `true`. When it is
`false`, describe the
published reference as inspectable but not executable by `play run`, show the reported blockers,
and recommend the pull command plus the compatible local runner returned by the push result.

Eligibility and inspection are not execution verification. For an eligible Play, resolve
representative parameters from the play's tested release contract. Present the inspection summary
and the exact pinned command without `--yes` before every acceptance run. For an interactive human,
run that command unchanged and let `play run` ask before setup. For an agent or other non-interactive
runner, obtain the user's explicit approval for that exact run before adding `--yes`; never infer
approval merely because the inputs or effects appear safe. After approval, use `--yes` only to carry
that consent across the non-interactive boundary:

```bash
rote play run <resolved-reference> <name=value...> --yes
```

`--yes` skips only the initial play/setup confirmation. It does not bypass adapter selection,
credential acquisition, provider OAuth, or runtime security checks. Preserve the emitted
remediation when one of those later gates stops the run, change the relevant state, and only then
retry.

Never automate the interactive `Ready` selector by piping `y`, `yes`, or any other input into
`play run`. A pipe makes stdin non-interactive and must fail closed. A headless harness carries
approval only by adding `--yes` after it has presented the exact parameters and obtained the user's
explicit approval.

Treat this published-reference run as the final acceptance test; lint, release, push, and
`play inspect` do not substitute for it. If explicit approval or required inputs are unavailable,
return `execution_verification: unverified`, state why, and never describe the Play as
execution-verified or successfully playable. On success, return
`execution_verification: passed` with the exact command and result summary. On failure, return
`execution_verification: failed` with the error and keep the Play out of the verified handoff.

Qualify resolution and execution separately, and take the statement from the push result rather than
deriving one from visibility. Who can resolve a Play and who can run it are different questions about
the same artifact: a public flow that declares process or browser privileges is resolvable by anyone
and runnable only by its owner or authorized org members, so any sentence composed from visibility
alone states the wrong execution audience. The push result carries the authored access guidance for
that pair; present it as given and add only the sharing mechanics below. Keep the full typed
verification packet in the agent handoff; the user needs the usable reference and relevant access
or execution limits.

- **Public** — share the canonical `play_uri`; anyone can GET its transparency card.
- **Private** — the canonical URI reveals its owner and slug even though unauthorized resolution
  remains indistinguishable from not-found. Prefer an opaque, revocable share URI when the registry
  returns one; possession identifies the share but never replaces Google/GitHub identity and
  current organization membership. For an org-owned play, continue to Stage 4 to offer an
  invitation before sharing.

Include the Play location, any Play URI and bootstrap transition, resolved run reference, execution
readiness, blockers, execution-verification status and evidence, and access guidance in the return
data. Continue to Stage 4 only for an organization-owned play. Do not construct a URL, parse it out
of a command, or hardcode
a play host; the play-push result owns the canonical Play and bootstrap URIs, and `rote play inspect` owns the
resolved run reference and execution assessment.

If inspection or the acceptance run fails, report the error and do not claim that the Play was
verified. A local release that was not published has no published Play URI.

An uninstalled registry result cannot go straight to authoring because fork accepts only a local
numbered source. When `rote-flow-run` returns a behavior-changing result with
`installation_state: uninstalled`, this skill owns the registry-to-local transition. Require an
inspected, exact `<org>/<name>@<version>` reference and approval to install that version. Apply the
same transition when direct authoring returns an uninstalled registry reference. Then run:

```bash
rote registry play pull <org>/<name>@<version> --yes
```

Hand off to `rote-flow-authoring` and name `rote guidance play forking` only after pull reports the
exact managed identity and local path. The handoff is complete when that exact version is installed
and selectable for authoring. Return `installation_state: exact_local` with the installed identity
and path. If pull fails, keep ownership here and return its blocker.

---

## Stage V — Change published visibility in place

Use this path when the adapter or play is already published and the user wants to change who can
discover or pull it. Confirm the exact `<org/name>` and target visibility if the user did not supply
both, then run the matching command:

```bash
rote registry adapter visibility <org/name> <public|private> --json
rote registry play visibility <org/name> <public|private> --json
```

- The change is **bidirectional** (`public` → `private` and `private` → `public`) and happens in
  place; no re-push or delete is needed, and version history and download counts are preserved.
- The command is **idempotent**. Asking for the current visibility succeeds and reports
  `already <visibility>` instead of erroring.
- A flip toward a visibility tier at capacity fails with `quota_exceeded` in either direction,
  including `public` → `private` when the private tier is full. Report the error as-is and use
  Stage U to show the current quota and limit.
- A not-found response deliberately says `not found (or you are not authorized to change its
  visibility)`. Report it as-is and do not probe other registry surfaces to reveal whether the
  artifact exists.
- A private flip changes registry access immediately, including future pinned or dependency-driven
  pulls, but already-pulled local copies are unaffected.

On success, report the canonical artifact reference and either `<old> -> <new>` or
`already <visibility>`. For automation, prefer the boolean `data.result.changed` (`true` means the
visibility changed; `false` means the request was an idempotent no-op) instead of string-matching
the human `->` or `already` line.

---

## Stage 4 — Turn the push into collaboration (members + invite)

After publication to an organization, offer to invite others to that organization so they can
review or use the new artifact. The play is: **ask → snapshot who's already there → collect a set
of emails → dedup → pick role(s) → invite each sequentially → report**.

### 4a — Ask first

Ask: **invite anyone to `<slug>` to review/use `<artifact>`?**
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

The role determines what the invitee can do. Present all three choices (default
**developer**) and either apply one role to the whole batch or let the user set it per email:

| Role | Can do | Use for |
|------|--------|---------|
| **reader** | Pull/use artifacts in the org. Cannot push or manage. | Teammates who only consume plays/adapters. |
| **developer** | reader **+ push/update** adapters and plays. Cannot manage members or org settings. | Collaborators who contribute artifacts. |
| **admin** | developer **+ manage the org**: invite/remove members, change roles, manage artifacts and visibility. | Co-maintainers you fully trust. |

> **Admin caveat — call this out before granting it.** An admin can invite and remove *other*
> members (including downgrading peers), change anyone's role, and delete/re-publish the org's
> artifacts. It's near-total control short of deleting the org itself (owner-only). Grant it only
> to people you'd trust to run the org; for someone who just needs to contribute plays,
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
then call each slug once:
```bash
rote registry org usage <slug> --json
```
Usage requires the owner or admin role. On `permission_denied`, mark that org unavailable and
continue. Never retry the unchanged command.

The shape: `{ plan, current{members, public_adapters, private_adapters, public_flows,
private_flows}, limit{... null = unlimited}, features{audit, verification} }`.

Render a per-org table — for each metric show `current / limit` (render `null` limit as
**∞ / unlimited**), the plan, and which feature flags are on. Flag anything near a non-null
limit (e.g. ≥80%) so the user sees pressure before they hit it. Example row:

> **conikee-home** · plan `org_admin` · members 1/∞ · private adapters 3/∞ · private plays
> 2/∞ · public 0 · features: audit ✓, verification ✓

This is the usage answer — quota consumption at a glance, no surprises.

---

## Notes

- **Don't re-push identical content.** The Stage 2 fingerprint/version check exists to protect
  quota — surfacing "already in sync" is a feature, not a non-answer.
- **`adapter publish` vs `adapter push`:** `publish` packs then pushes (the normal path);
  `push` uploads an already-packed `.adapt`. Prefer `publish` for a freshly-minted adapter.
- **Play drafts:** resume `rote-flow-authoring` for validation before preparing a snapshot for
  publication. A saved local development copy is a complete local result.
- For deeper org administration (create/delete orgs, change roles, remove members, manage
  pending invites at length), hand off to the **rote-org** skill — this skill covers the
  push-time slice (members + invite); rote-org is the full admin surface.

---

## Closing line + related skills

**Closing line** (only after a clean push): one dry one-liner keyed to what landed — e.g. the
artifact name + org + that it's now shareable without a middleman registry tax. Skip it if the push
errored or there was nothing to push.

**Related setup skills:**
- **Invoked by:** `rote-adapter-create` (after minting an adapter) and the play
  crystallize path.
- **Full org admin:** the **rote-org** skill · **Tune the adapter first:**
  `rote-adapter-config`
